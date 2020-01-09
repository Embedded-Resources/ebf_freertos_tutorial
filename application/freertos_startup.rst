.. vim: syntax=rst

FreeRTOS的启动流程
====================

在目前的RTOS中，主要有两种比较流行的启动方式，暂时还没有看到第三种，接下来我将通过伪代码的方式来讲解下这两种启动方式的区别，然后再具体分析下FreeRTOS的启动流程。

万事俱备，只欠东风
~~~~~~~~~

第一种我称之为万事俱备，只欠东风法。这种方法是在main函数中将硬件初始化，RTOS系统初始化，所有任务的创建这些都弄好，这个我称之为万事都已经准备好。最后只欠一道东风，即启动RTOS的调度器，开始多任务的调度，具体的伪代码实现见代码清单13‑1。

代码清单13‑1万事俱备，只欠东风法伪代码实现

1 int main (void)

2 {

3 /\* 硬件初始化 \*/

4 HardWare_Init();\ **(1)**

5

6 /\* RTOS 系统初始化 \*/

7 RTOS_Init();\ **(2)**

8

9 /\* 创建任务1，但任务1不会执行，因为调度器还没有开启 \*/**(3)**

10 RTOS_TaskCreate(Task1);

11 /\* 创建任务2，但任务2不会执行，因为调度器还没有开启 \*/

12 RTOS_TaskCreate(Task2);

13

14 /\* ......继续创建各种任务 \*/

15

16 /\* 启动RTOS，开始调度 \*/

17 RTOS_Start();\ **(4)**

18 }

19

20 voidTask1( void \*arg )\ **(5)**

21 {

22 while (1)

23 {

24 /\* 任务实体，必须有阻塞的情况出现 \*/

25 }

26 }

27

28 voidTask1( void \*arg )\ **(6)**

29 {

30 while (1)

31 {

32 /\* 任务实体，必须有阻塞的情况出现 \*/

33 }

34 }

代码清单13‑1\ **(1)**\ ：硬件初始化。硬件初始化这一步还属于裸机的范畴，我们可以把需要使用到的硬件都初始化好而且测试好，确保无误。

代码清单13‑1\ **(2)**\ ：RTOS系统初始化。比如RTOS里面的全局变量的初始化，空闲任务的创建等。不同的RTOS，它们的初始化有细微的差别。

代码清单13‑1\ **(3)**\ ：创建各种任务。这里把所有要用到的任务都创建好，但还不会进入调度，因为这个时候RTOS的调度器还没有开启。

代码清单13‑1\ **(4)**\ ：启动RTOS调度器，开始任务调度。这个时候调度器就从刚刚创建好的任务中选择一个优先级最高的任务开始运行。

代码清单13‑1\ **(5)(6)**\ ：任务实体通常是一个不带返回值的无限循环的C函数，函数体必须有阻塞的情况出现，不然任务（如果优先权恰好是最高）会一直在while循环里面执行，导致其他任务没有执行的机会。

小心翼翼，十分谨慎
~~~~~~~~~

第二种我称之为小心翼翼，十分谨慎法。这种方法是在main函数中将硬件和RTOS系统先初始化好，然后创建一个启动任务后就启动调度器，然后在启动任务里面创建各种应用任务，当所有任务都创建成功后，启动任务把自己删除，具体的伪代码实现见代码清单13‑2。

代码清单13‑2小心翼翼，十分谨慎法伪代码实现

1 int main (void)

2 {

3 /\* 硬件初始化 \*/

4 HardWare_Init();\ **(1)**

5

6 /\* RTOS 系统初始化 \*/

7 RTOS_Init();\ **(2)**

8

9 /\* 创建一个任务 \*/

10 RTOS_TaskCreate(AppTaskCreate);\ **(3)**

11

12 /\* 启动RTOS，开始调度 \*/

13 RTOS_Start();\ **(4)**

14 }

15

16 /\* 起始任务，在里面创建任务 \*/

17 voidAppTaskCreate( void \*arg )\ **(5)**

18 {

19 /\* 创建任务1，然后执行 \*/

20 RTOS_TaskCreate(Task1);\ **(6)**

21

22 /\* 当任务1阻塞时，继续创建任务2，然后执行 \*/

23 RTOS_TaskCreate(Task2);

24

25 /\* ......继续创建各种任务 \*/

26

27 /\* 当任务创建完成，删除起始任务 \*/

28 RTOS_TaskDelete(AppTaskCreate);\ **(7)**

29 }

30

31 void Task1( void \*arg )\ **(8)**

32 {

33 while (1)

34 {

35 /\* 任务实体，必须有阻塞的情况出现 \*/

36 }

37 }

38

39 void Task2( void \*arg )\ **(9)**

40 {

41 while (1)

42 {

43 /\* 任务实体，必须有阻塞的情况出现 \*/

44 }

45 }

代码清单13‑2 **(1)**\ ：硬件初始化。来到硬件初始化这一步还属于裸机的范畴，我们可以把需要使用到的硬件都初始化好而且测试好，确保无误。

代码清单13‑2 **(2)**\ ：RTOS系统初始化。比如RTOS里面的全局变量的初始化，空闲任务的创建等。不同的RTOS，它们的初始化有细微的差别。

代码清单13‑2 **(3)**\ ：创建一个开始任务。然后在这个初始任务里面创建各种应用任务。

代码清单13‑2 **(4)**\ ：启动RTOS调度器，开始任务调度。这个时候调度器就去执行刚刚创建好的初始任务。

代码清单13‑2 **(5)**\ ：我们通常说任务是一个不带返回值的无限循环的C函数，但是因为初始任务的特殊性，它不能是无限循环的，只执行一次后就关闭。在初始任务里面我们创建我们需要的各种任务。

代码清单13‑2 **(6)**\ ：创建任务。每创建一个任务后它都将进入就绪态，系统会进行一次调度，如果新创建的任务的优先级比初始任务的优先级高的话，那将去执行新创建的任务，当新的任务阻塞时再回到初始任务被打断的地方继续执行。反之，则继续往下创建新的任务，直到所有任务创建完成。

代码清单13‑2 **(7)**\ ：各种应用任务创建完成后，初始任务自己关闭自己，使命完成。

代码清单13‑2 **(8)(9)**\ ：任务实体通常是一个不带返回值的无限循环的C函数，函数体必须有阻塞的情况出现，不然任务（如果优先权恰好是最高）会一直在while循环里面执行，其他任务没有执行的机会。

孰优孰劣
~~~~

那有关这两种方法孰优孰劣？我暂时没发现，我个人还是比较喜欢使用第一种。LiteOS和ucos第一种和第二种都可以使用，由用户选择，RT-Thread和FreeRTOS则默认使用第二种。接下来我们详细讲解下FreeRTOS的启动流程。

.. _freertos的启动流程-1:

FreeRTOS的启动流程
~~~~~~~~~~~~~

我们知道，在系统上电的时候第一个执行的是启动文件里面由汇编编写的复位函数Reset_Handler，具体见代码清单13‑3。复位函数的最后会调用C库函数__main，具体见代码清单13‑3的加粗部分。__main函数的主要工作是初始化系统的堆和栈，最后调用C中的main函数，从而去到C的世界。

代码清单13‑3Reset_Handler函数

1 Reset_Handler PROC

2 EXPORT Reset_Handler [WEAK]

3 IMPORT \__main

4 IMPORT SystemInit

5 LDRR0, =SystemInit

6 BLX R0

7 LDRR0, =__main

8 BX R0

9 ENDP

创建任务xTaskCreate()函数
^^^^^^^^^^^^^^^^^^^

在main()函数中，我们直接可以对FreeRTOS进行创建任务操作，因为FreeRTOS会自动帮我们做初始化的事情，比如初始化堆内存。FreeRTOS的简单方便是在别的实时操作系统上都没有的，像RT-Tharead，需要做很多事情，具体可以看野火出版的另一本书《RT-
Thread内核实现与应用开发实战—基于STM32》；华为LiteOS也需要我们用户进行初始化内核，具体可以看野火出版的另一本书籍华为LiteOS《华为LiteOS内核实现与应用开发实战—基于STM32》。

这种简单的特点使得FreeRTOS在初学的时候变得很简单，我们自己在main()函数中直接初始化我们的板级外设——BSP_Init()，然后进行任务的创建即可——xTaskCreate()，在任务创建中，FreeRTOS会帮我们进行一系列的系统初始化，在创建任务的时候，会帮我们初始化堆内存，具体见代
码清单13‑4。

代码清单13‑4 xTaskCreate函数内部进行堆内存初始化

1 BaseType_t xTaskCreate(TaskFunction_t pxTaskCode,

2 const char \* const pcName,

3 const uint16_t usStackDepth,

4 void \* const pvParameters,

5 UBaseType_t uxPriority,

6 TaskHandle_t \* const pxCreatedTask )

7 {

8 if ( pxStack != NULL ) {

**9 /\* 分配任务控制块内存 \*/**

**10 pxNewTCB = ( TCB_t \* ) pvPortMalloc( sizeof( TCB_t ) ); (1)**

11

12

13 if ( pxNewTCB != NULL ) {

14 /*将栈位置存储在TCB中。*/

15 pxNewTCB->pxStack = pxStack;

16 }

17 }

18 /\*

19 省略代码

20 ......

21 \*/

22 }

23

24 **/\* 分配内存函数 \*/**

**25 void \*pvPortMalloc( size_t xWantedSize )**

26 {

27 BlockLink_t \*pxBlock, \*pxPreviousBlock, \*pxNewBlockLink;

28 void \*pvReturn = NULL;

29

30 vTaskSuspendAll();

31 {

32

33 **/*如果这是对malloc的第一次调用，那么堆将需要初始化来设置空闲块列表。*/**

34 if ( pxEnd == NULL ) {

**35 prvHeapInit(); (2)**

36 } else {

37 mtCOVERAGE_TEST_MARKER();

38 }

39 /\*

40 省略代码

41 ......

42 \*/

43

44 }

45 }

从代码清单13‑4的\ **(1)(2)**\ 中，我们知道：在未初始化内存的时候一旦调用了xTaskCreate()函数，FreeRTOS就会帮我们自动进行内存的初始化，内存的初始化具体见代码清单13‑5。注意，此函数是FreeRTOS内部调用的，目前我们暂时不用管这个函数的实现，在后面我们会仔细
讲解FreeRTOS的内存管理相关知识，现在我们知道FreeRTOS会帮我们初始话系统要用的东西即可。

代码清单13‑5prvHeapInit()函数定义

1 static void prvHeapInit( void )

2 {

3 BlockLink_t \*pxFirstFreeBlock;

4 uint8_t \*pucAlignedHeap;

5 size_t uxAddress;

6 size_t xTotalHeapSize = configTOTAL_HEAP_SIZE;

7

8

9 uxAddress = ( size_t ) ucHeap;

10 /*确保堆在正确对齐的边界上启动。*/

11 if ( ( uxAddress & portBYTE_ALIGNMENT_MASK ) != 0 ) {

12 uxAddress += ( portBYTE_ALIGNMENT - 1 );

13 uxAddress &= ~( ( size_t ) portBYTE_ALIGNMENT_MASK );

14 xTotalHeapSize -= uxAddress - ( size_t ) ucHeap;

15 }

16

17 pucAlignedHeap = ( uint8_t \* ) uxAddress;

18

19 /\* xStart用于保存指向空闲块列表中第一个项目的指针。

20 void用于防止编译器警告*/

21 xStart.pxNextFreeBlock = ( void \* ) pucAlignedHeap;

22 xStart.xBlockSize = ( size_t ) 0;

23

24

25 /\* pxEnd用于标记空闲块列表的末尾，并插入堆空间的末尾。*/

26 uxAddress = ( ( size_t ) pucAlignedHeap ) + xTotalHeapSize;

27 uxAddress -= xHeapStructSize;

28 uxAddress &= ~( ( size_t ) portBYTE_ALIGNMENT_MASK );

29 pxEnd = ( void \* ) uxAddress;

30 pxEnd->xBlockSize = 0;

31 pxEnd->pxNextFreeBlock = NULL;

32

33

34 /*首先，有一个空闲块，其大小可以占用整个堆空间，减去pxEnd占用的空间。*/

35 pxFirstFreeBlock = ( void \* ) pucAlignedHeap;

36 pxFirstFreeBlock->xBlockSize = uxAddress - ( size_t ) pxFirstFreeBlock;

37 pxFirstFreeBlock->pxNextFreeBlock = pxEnd;

38

39 /*只存在一个块 - 它覆盖整个可用堆空间。因为是刚初始化的堆内存*/

40 xMinimumEverFreeBytesRemaining = pxFirstFreeBlock->xBlockSize;

41 xFreeBytesRemaining = pxFirstFreeBlock->xBlockSize;

42

43

44 xBlockAllocatedBit = ( ( size_t ) 1 ) << ( ( sizeof( size_t ) \*

45 heapBITS_PER_BYTE ) - 1 );

46}

47/*-----------------------------------------------------------*/

vTaskStartScheduler()函数
^^^^^^^^^^^^^^^^^^^^^^^

在创建完任务的时候，我们需要开启调度器，因为创建仅仅是把任务添加到系统中，还没真正调度，并且空闲任务也没实现，定时器任务也没实现，这些都是在开启调度函数vTaskStartScheduler()中实现的。为什么要空闲任务？因为FreeRTOS一旦启动，就必须保证系统中每时每刻都有一个任务处于运行态（
Runing），并且空闲任务不可以被挂起与删除，空闲任务的优先级是最低的，以便系统中其他任务能随时抢占空闲任务的CPU使用权。这些都是系统必要的东西，也无需用户自己实现，FreeRTOS全部帮我们搞定了。处理完这些必要的东西之后，系统才真正开始启动，具体见代码清单13‑6加粗部分。

代码清单13‑6vTaskStartScheduler()函数

1 /*-----------------------------------------------------------*/

2

3 void vTaskStartScheduler( void )

4 {

5 BaseType_t xReturn;

6

7 /*添加空闲任务*/

8 #if( configSUPPORT_STATIC_ALLOCATION == 1 )

9 {

10 StaticTask_t \*pxIdleTaskTCBBuffer = NULL;

11 StackType_t \*pxIdleTaskStackBuffer = NULL;

12 uint32_t ulIdleTaskStackSize;

13

14 /\* 空闲任务是使用用户提供的RAM创建的 - 获取

15 然后RAM的地址创建空闲任务。这是静态创建任务，我们不用管*/

16 vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer,

17 &pxIdleTaskStackBuffer,

18 &ulIdleTaskStackSize );

19 xIdleTaskHandle = xTaskCreateStatic(prvIdleTask,

20 "IDLE",

21 ulIdleTaskStackSize,

22 ( void \* ) NULL,

23 ( tskIDLE_PRIORITY \| portPRIVILEGE_BIT ),

24 pxIdleTaskStackBuffer,

25 pxIdleTaskTCBBuffer );

26

27 if ( xIdleTaskHandle != NULL ) {

28 xReturn = pdPASS;

29 } else {

30 xReturn = pdFAIL;

31 }

32 }

33 #else /\* 这里才是动态创建idle任务 \*/

34 {

**35 /\* 使用动态分配的RAM创建空闲任务。 \*/**

**36 xReturn = xTaskCreate( prvIdleTask, (1)**

**37 "IDLE", configMINIMAL_STACK_SIZE,**

**38 ( void \* ) NULL,**

**39 ( tskIDLE_PRIORITY \| portPRIVILEGE_BIT ),**

**40 &xIdleTaskHandle );**

41 }

42 #endif

43

44 #if ( configUSE_TIMERS == 1 )

45 {

**46 /\* 如果使能了 configUSE_TIMERS宏定义**

**47 表明使用定时器，需要创建定时器任务*/**

**48 if ( xReturn == pdPASS ) {**

**49 xReturn = xTimerCreateTimerTask(); (2)**

50 } else {

51 mtCOVERAGE_TEST_MARKER();

52 }

53 }

54 #endif/\* configUSE_TIMERS \*/

55

56 if ( xReturn == pdPASS ) {

57 /\* 此处关闭中断，以确保不会发生中断

58 在调用xPortStartScheduler（）之前或期间。栈的

59 创建的任务包含打开中断的状态

60 因此，当第一个任务时，中断将自动重新启用

61 开始运行。 \*/

62 portDISABLE_INTERRUPTS();

63

64 #if ( configUSE_NEWLIB_REENTRANT == 1 )

65 {

66 /\* 不需要理会，这个宏定义没打开 \*/

67 \_impure_ptr = &( pxCurrentTCB->xNewLib_reent );

68 }

69 #endif/\* configUSE_NEWLIB_REENTRANT \*/

70

71 xNextTaskUnblockTime = portMAX_DELAY;

72 xSchedulerRunning = pdTRUE; **(3)**

73 xTickCount = ( TickType_t ) 0U;

74

75 /\* 如果定义了configGENERATE_RUN_TIME_STATS，则以下内容

76 必须定义宏以配置用于生成的计时器/计数器

77 运行时计数器时基。目前没启用该宏定义 \*/

78 portCONFIGURE_TIMER_FOR_RUN_TIME_STATS();

79

80 /\* 调用xPortStartScheduler函数配置相关硬件

81 如滴答定时器、FPU、pendsv等 \*/

**82 if ( xPortStartScheduler() != pdFALSE ) { (4)**

83 /\* 如果xPortStartScheduler函数启动成功，则不会运行到这里 \*/

84 } else {

85 /\* 不会运行到这里，除非调用 xTaskEndScheduler() 函数 \*/

86 }

87 } else {

88 /\* 只有在内核无法启动时才会到达此行，

89 因为没有足够的堆内存来创建空闲任务或计时器任务。

90 此处使用了断言，会输出错误信息，方便错误定位 \*/

91 configASSERT( xReturn != errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY );

92 }

93

94 /\* 如果INCLUDE_xTaskGetIdleTaskHandle设置为0，则防止编译器警告，

95 这意味着在其他任何地方都不使用xIdleTaskHandle。暂时不用理会 \*/

96 ( void ) xIdleTaskHandle;

97 }

98 /*-----------------------------------------------------------*/

代码清单13‑6\ **(1)**\ ：动态创建空闲任务（IDLE），因为现在我们不使用静态创建，这个configSUPPORT_STATIC_ALLOCATION宏定义为0，只能是动态创建空闲任务，并且空闲任务的优先级与栈大小都在FreeRTOSConfig.h中由用户定义，空闲任务的任务句柄存放
在静态变量xIdleTaskHandle中，用户可以调用API函数xTaskGetIdleTaskHandle()获得空闲任务句柄。

代码清单13‑6\ **(2)**\ ：如果在FreeRTOSConfig.h中使能了configUSE_TIMERS这个宏定义，那么需要创建一个定时器任务，这个定时器任务也是调用xTaskCreate()函数完成创建，过程十分简单，这也是系统的初始化内容，在调度器启动的过程中发现必要初始化的东西，
FreeRTOS就会帮我们完成，真的对开发者太友好了，xTimerCreateTimerTask()函数具体见代码清单13‑7加粗部分。

代码清单13‑7xTimerCreateTimerTask源码

1 BaseType_t xTimerCreateTimerTask( void )

2 {

3 BaseType_t xReturn = pdFAIL;

4

5 /\* 检查使用了哪些活动计时器的列表，以及

6 用于与计时器服务通信的队列，已经

7 初始化。*/

8 prvCheckForValidListAndQueue();

9

10 if ( xTimerQueue != NULL ) {

11 #if( configSUPPORT_STATIC_ALLOCATION == 1 )

12 {

13 /\* 这是静态创建的，无需理会 \*/

14 StaticTask_t \*pxTimerTaskTCBBuffer = NULL;

15 StackType_t \*pxTimerTaskStackBuffer = NULL;

16 uint32_t ulTimerTaskStackSize;

17

18 vApplicationGetTimerTaskMemory(&pxTimerTaskTCBBuffer,

19 &pxTimerTaskStackBuffer,

20 &ulTimerTaskStackSize );

21 xTimerTaskHandle = xTaskCreateStatic(prvTimerTask,

22 "Tmr Svc",

23 ulTimerTaskStackSize,

24 NULL,

25 ( ( UBaseType_t ) configTIMER_TASK_PRIORITY ) \|

26 portPRIVILEGE_BIT,

27 pxTimerTaskStackBuffer,

28 pxTimerTaskTCBBuffer );

29

30 if ( xTimerTaskHandle != NULL )

31 {

32 xReturn = pdPASS;

33 }

34 }

**35 #else**

**36 {/\* 这是才是动态创建定时器任务*/**

**37 xReturn = xTaskCreate(prvTimerTask,**

**38 "Tmr Svc",**

**39 configTIMER_TASK_STACK_DEPTH,**

**40 NULL,**

**41 ( ( UBaseType_t ) configTIMER_TASK_PRIORITY ) \|**

**42 portPRIVILEGE_BIT,**

**43 &xTimerTaskHandle );**

**44 }**

45 #endif/\* configSUPPORT_STATIC_ALLOCATION \*/

46 } else {

47 mtCOVERAGE_TEST_MARKER();

48 }

49

50 configASSERT( xReturn );

51 return xReturn;

52 }

代码清单13‑6\ **(3)**\ ：xSchedulerRunning等于pdTRUE，表示调度器开始运行了，而xTickCount初始化需要初始化为0，这个xTickCount变量用于记录系统的时间，在节拍定时器（SysTick）中断服务函数中进行自加。

代码清单13‑6\ **(4)**\ ：调用函数xPortStartScheduler()来启动系统节拍定时器（一般都是使用SysTick）并启动第一个任务。因为设置系统节拍定时器涉及硬件特性，因此函数xPortStartScheduler()由移植层提供（在port.c文件实现），不同的硬件架构，
这个函数的代码也不相同，在ARM_CM3中，使用SysTick作为系统节拍定时器。有兴趣可以看看xPortStartScheduler()的源码内容，下面我只是简单介绍一下相关知识。

在Cortex-M3架构中，FreeRTOS为了任务启动和任务切换使用了三个异常：SVC、PendSV和SysTick：

SVC（系统服务调用，亦简称系统调用）用于任务启动，有些操作系统不允许应用程序直接访问硬件，而是通过提供一些系统服务函数，用户程序使用 SVC 发出对系统服务函数的呼叫请求，以这种方法调用它们来间接访问硬件，它就会产生一个 SVC 异常。

PendSV（可挂起系统调用）用于完成任务切换，它是可以像普通的中断一样被挂起的，它的最大特性是如果当前有优先级比它高的中断在运行，PendSV会延迟执行，直到高优先级中断执行完毕，这样子产生的PendSV中断就不会打断其他中断的运行。

SysTick用于产生系统节拍时钟，提供一个时间片，如果多个任务共享同一个优先级，则每次SysTick中断，下一个任务将获得一个时间片。关于详细的SVC、PendSV异常描述，推荐《Cortex-M3权威指南》一书的“异常”部分。

这里将PendSV和SysTick异常优先级设置为最低，这样任务切换不会打断某个中断服务程序，中断服务程序也不会被延迟，这样简化了设计，有利于系统稳定。有人可能会问，那SysTick的优先级配置为最低，那延迟的话系统时间会不会有偏差？答案是不会的，因为SysTick只是当次响应中断被延迟了，而Sys
Tick是硬件定时器，它一直在计时，这一次的溢出产生中断与下一次的溢出产生中断的时间间隔是一样的，至于系统是否响应还是延迟响应，这个与SysTick无关，它照样在计时。

main函数
^^^^^^

当我们拿到一个移植好FreeRTOS的例程的时候，不出意外，你首先看到的是main函数，当你认真一看main函数里面只是创建并启动一些任务和硬件初始化，具体见代码清单13‑8。而系统初始化这些工作不需要我们实现，因为FreeRTOS在我们使用创建与开启调度的时候就已经偷偷帮我们做完了，如果只是使用F
reeRTOS的话，无需关注FreeRTOS API函数里面的实现过程，但是我们还是建议需要深入了解FreeRTOS然后再去使用，避免出现问题。

代码清单13‑8 main函数

1 /\*

2 \* @brief 主函数

3 \* @param 无

4 \* @retval 无

5 \* @note 第一步：开发板硬件初始化

6 第二步：创建APP应用任务

7 第三步：启动FreeRTOS，开始多任务调度

8 \/

9 int main(void)

10 {

11 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

12

13 /\* 开发板硬件初始化 \*/

14 BSP_Init(); **(1)**

15 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS-多任务创建实验!\r\n");

16 /\* 创建AppTaskCreate任务 \*/ **(2)**

17 xReturn = **xTaskCreate**\ ((TaskFunction_t )AppTaskCreate,/\* 任务入口函数 \*/

18 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

19 (uint16_t )512, /\* 任务栈大小 \*/

20 (void\* )NULL,/\* 任务入口函数参数 \*/

21 (UBaseType_t )1, /\* 任务的优先级 \*/

22 (TaskHandle_t*)&AppTaskCreate_Handle);/*任务控制块指针*/

23 /\* 启动任务调度 \*/

24 if (pdPASS == xReturn)

25 **vTaskStartScheduler**\ (); /\* 启动任务，开启调度 \*/ **(3)**

26 else

27 return -1; **(4)**

28

29 while (1); /\* 正常不会执行到这里 \*/

30 }

代码清单13‑8\ **(1)**\ ：开发板硬件初始化，FreeRTOS系统初始化是经在创建任务与开启调度器的时候完成的。

代码清单13‑8\ **(2)**\ ：在AppTaskCreate中创建各种应用任务，具体见代码清单13‑9。

代码清单13‑9 AppTaskCreate函数

1 /\*

2 \* @ 函数名： AppTaskCreate

3 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

4 \* @ 参数：无

5 \* @ 返回值：无

6 \/

7 static void AppTaskCreate(void)

8 {

9 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

10

11 taskENTER_CRITICAL(); //进入临界区

12

13 /\* 创建LED_Task任务 \*/

14 xReturn = xTaskCreate((TaskFunction_t )LED1_Task, /\* 任务入口函数 \*/

15 (const char\* )"LED1_Task",/\* 任务名字 \*/

16 (uint16_t )512, /\* 任务栈大小 \*/

17 (void\* )NULL, /\* 任务入口函数参数 \*/

18 (UBaseType_t )2, /\* 任务的优先级 \*/

19 (TaskHandle_t\* )&LED1_Task_Handle);/\* 任务控制块指针 \*/

20 if (pdPASS == xReturn)

21 printf("创建LED1_Task任务成功!\r\n");

22

23 /\* 创建LED_Task任务 \*/

24 xReturn = xTaskCreate((TaskFunction_t )LED2_Task, /\* 任务入口函数 \*/

25 (const char\* )"LED2_Task",/\* 任务名字 \*/

26 (uint16_t )512, /\* 任务栈大小 \*/

27 (void\* )NULL, /\* 任务入口函数参数 \*/

28 (UBaseType_t )3, /\* 任务的优先级 \*/

29 (TaskHandle_t\* )&LED2_Task_Handle);/\* 任务控制块指针 \*/

30 if (pdPASS == xReturn)

31 printf("创建LED2_Task任务成功!\r\n");

32

33 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

34

35 taskEXIT_CRITICAL(); //退出临界区

36 }

当创建的应用任务的优先级比AppTaskCreate任务的优先级高、低或者相等时候，程序是如何执行的？假如像我们代码一样在临界区创建任务，任务只能在退出临界区的时候才执行最高优先级任务。假如没使用临界区的话，就会分三种情况：1、应用任务的优先级比初始任务的优先级高，那创建完后立马去执行刚刚创建的应用
任务，当应用任务被阻塞时，继续回到初始任务被打断的地方继续往下执行，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命；2、应用任务的优先级与初始任务的优先级一样，那创建完后根据任务的时间片来执行，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命；3、应用任务的优先级比
初始任务的优先级低，那创建完后任务不会被执行，如果还有应用任务紧接着创建应用任务，如果应用任务的优先级出现了比初始任务高或者相等的情况，请参考1和2的处理方式，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命。

代码清单13‑8\ **(3)(4)**\ ：在启动任务调度器的时候，假如启动成功的话，任务就不会有返回了，假如启动没成功，则通过LR寄存器指定的地址退出，在创建AppTaskCreate任务的时候，任务栈对应LR寄存器指向是任务退出函数prvTaskExitError()，该函数里面是一个死循环，
这代表着假如创建任务没成功的话，就会进入死循环，该任务也不会运行。
