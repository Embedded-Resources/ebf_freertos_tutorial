.. vim: syntax=rst

任务通知
----

任务通知的基本概念
~~~~~~~~~

FreeRTOS 从V8.2.0版本开始提供任务通知这个功能，每个任务都有一个32位的通知值，在大多数情况下，任务通知可以替代二值信号量、计数信号量、事件组，也可以替代长度为1的队列（可以保存一个32位整数或指针值）。

相对于以前使用FreeRTOS内核通信的资源，必须创建队列、二进制信号量、计数信号量或事件组的情况，使用任务通知显然更灵活。按照 FreeRTOS 官方的说法，使用任务通知比通过信号量等ICP通信方式解除阻塞的任务要快
45%，并且更加省RAM内存空间（使用GCC编译器，-o2优化级别），任务通知的使用无需创建队列。想要使用任务通知，必须将FreeRTOSConfig.h中的宏定义configUSE_TASK_NOTIFICATIONS设置为1，其实FreeRTOS默认是为1的，所以任务通知是默认使能的。

FreeRTOS 提供以下几种方式发送通知给任务：

-  发送通知给任务，如果有通知未读，不覆盖通知值。

-  发送通知给任务，直接覆盖通知值。

-  发送通知给任务，设置通知值的一个或者多个位，可以当做事件组来使用。

-  发送通知给任务，递增通知值，可以当做计数信号量使用。

通过对以上任务通知方式的合理使用，可以在一定场合下替代FreeRTOS的信号量，队列、事件组等。

当然，凡是都有利弊，不然的话FreeRTOS还要内核的IPC通信机制干嘛，消息通知虽然处理更快，RAM开销更小，但也有以下限制：

-  只能有一个任务接收通知消息，因为必须指定接收通知的任务。。

-  只有等待通知的任务可以被阻塞，发送通知的任务，在任何情况下都不会因为发送失败而进入阻塞态。

任务通知的运作机制
~~~~~~~~~

顾名思义，任务通知是属于任务中附带的资源，所以在任务被创建的时候，任务通知也被初始化的，而在分析队列和信号量的章节中，我们知道在使用队列、信号量前，必须先创建队列和信号量，目的是为了创建队列数据结构。比如使用xQueueCreate()函数创建队列，用xSemaphoreCreateBinary()
函数创建二值信号量等等。再来看任务通知，由于任务通知的数据结构包含在任务控制块中，只要任务存在，任务通知数据结构就已经创建完毕，可以直接使用，所以使用的时候很是方便。

任务通知可以在任务中向指定任务发送通知，也可以在中断中向指定任务发送通知，FreeRTOS的每个任务都有一个32位的通知值，任务控制块中的成员变量ulNotifiedValue就是这个通知值。只有在任务中可以等待通知，而不允许在中断中等待通知。如果任务在等待的通知暂时无效，任务会根据用户指定的阻塞超
时时间进入阻塞状态，我们可以将等待通知的任务看作是消费者；其他任务和中断可以向等待通知的任务发送通知，发送通知的任务和中断服务函数可以看作是生产者，当其他任务或者中断向这个任务发送任务通知，任务获得通知以后，该任务就会从阻塞态中解除，这与FreeRTOS中内核的其他通信机制一致。

任务通知的数据结构
~~~~~~~~~

从前文我们知道，任务通知是任务控制块的资源，那它也算任务控制块中的成员变量，包含在任务控制块中，我们将其拿出来看看，具体见代码清单20‑1加粗部分。

代码清单20‑1任务控制块中的任务通知成员变量

1 typedefstruct tskTaskControlBlock {

2 volatile StackType_t \*pxTopOfStack;

3

4 #if ( portUSING_MPU_WRAPPERS == 1 )

5 xMPU_SETTINGS xMPUSettings;

6 #endif

7

8 ListItem_t xStateListItem;

9 ListItem_t xEventListItem;

10 UBaseType_t uxPriority;

11 StackType_t \*pxStack;

12 char pcTaskName[ configMAX_TASK_NAME_LEN ];

13

14 #if ( portSTACK_GROWTH > 0 )

15 StackType_t \*pxEndOfStack;

16 #endif

17

18 #if ( portCRITICAL_NESTING_IN_TCB == 1 )

19 UBaseType_t uxCriticalNesting;

20 #endif

21

22 #if ( configUSE_TRACE_FACILITY == 1 )

23 UBaseType_t uxTCBNumber;

24 UBaseType_t uxTaskNumber;

25 #endif

26

27 #if ( configUSE_MUTEXES == 1 )

28 UBaseType_t uxBasePriority;

29 UBaseType_t uxMutexesHeld;

30 #endif

31

32 #if ( configUSE_APPLICATION_TASK_TAG == 1 )

33 TaskHookFunction_t pxTaskTag;

34 #endif

35

36 #if( configNUM_THREAD_LOCAL_STORAGE_POINTERS > 0 )

37 void \*pvThreadLocalStoragePointers[ configNUM_THREAD_LOCAL_STORAGE_POINTERS ];

38 #endif

39

40 #if( configGENERATE_RUN_TIME_STATS == 1 )

41 uint32_t ulRunTimeCounter;

42 #endif

43

44 #if ( configUSE_NEWLIB_REENTRANT == 1 )

45 struct \_reent xNewLib_reent;

46 #endif

47

**48 #if( configUSE_TASK_NOTIFICATIONS == 1 )**

**49 volatileuint32_t ulNotifiedValue; (1)**

**50 volatileuint8_t ucNotifyState; (2)**

**51 #endif**

52

53 #if( tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0 )

54 uint8_t ucStaticallyAllocated;

55 #endif

56

57 #if( INCLUDE_xTaskAbortDelay == 1 )

58 uint8_t ucDelayAborted;

59 #endif

60

61 } tskTCB;

62

63 typedef tskTCB TCB_t;

代码清单20‑1\ **(1)**\ ：任务通知的值，可以保存一个32位整数或指针值。

代码清单20‑1\ **(2)**\ ：任务通知状态，用于标识任务是否在等待通知。

任务通知的函数接口讲解
~~~~~~~~~~~

发送任务通知函数xTaskGenericNotify()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^


我们先看一下发送通知API函数。这类函数比较多，有6个。但仔细分析会发现它们只能完成3种操作，每种操作有两个API函数，分别为带中断保护版本和不带中断保护版本。FreeRTOS将API细分为带中断保护版本和不带中断保护版本是为了节省中断服务程序处理时间，提升性能。通过前面通信机制的学习，相信大家都了
解了FreeRTOS的风格，这里的任务通知发送函数也是利用宏定义来进行扩展的，所有的函数都是一个宏定义，在任务中发送任务通知的函数均是调用xTaskGenericNotify()函数进行发送通知，下面来看看xTaskGenericNotify()的源码，具体见代码清单20‑2。

代码清单20‑2 xTaskGenericNotify()源码

1 #if( configUSE_TASK_NOTIFICATIONS == 1 )

2

3 BaseType_t xTaskGenericNotify( TaskHandle_t xTaskToNotify, **(1)**

4 uint32_t ulValue, **(2)**

5 eNotifyAction eAction, **(3)**

6 uint32_t \*pulPreviousNotificationValue ) **(4)**

7 {

8 TCB_t \* pxTCB;

9 BaseType_t xReturn = pdPASS;

10 uint8_t ucOriginalNotifyState;

11

12 configASSERT( xTaskToNotify );

13 pxTCB = ( TCB_t \* ) xTaskToNotify;

14

15 taskENTER_CRITICAL();

16 {

17 if ( pulPreviousNotificationValue != NULL ) {

18 /*回传未被更新的任务通知值*/

19 \*pulPreviousNotificationValue = pxTCB->ulNotifiedValue; **(5)**

20 }

21

22 /\* 获取任务通知的状态，看看任务是否在等待通知，方便在发送通知后恢复任务 \*/

23 ucOriginalNotifyState = pxTCB->ucNotifyState; **(6)**

24

25 /\* 不管状态是怎么样的，反正现在发送通知，任务就收到任务通知 \*/

26 pxTCB->ucNotifyState = taskNOTIFICATION_RECEIVED; **(7)**

27

28 /\* 指定更新任务通知的方式 \*/

29 switch ( eAction ) { **(8)**

30

31 /*通知值按位或上ulValue。

32 使用这种方法可以某些场景下代替事件组，但执行速度更快。*/

33 case eSetBits : **(9)**

34 pxTCB->ulNotifiedValue \|= ulValue;

35 break;

36

37 /\* 被通知任务的通知值增加1，这种发送通知方式，参数ulValue未使用 \*/

38 case eIncrement: **(10)**

39 ( pxTCB->ulNotifiedValue )++;

40 break;

41

42 /\* 将被通知任务的通知值设置为ulValue。无论任务是否还有通知，

43 都覆盖当前任务通知值。使用这种方法，

44 可以在某些场景下代替xQueueoverwrite()函数，但执行速度更快。 \*/

45 case eSetValueWithOverwrite: **(11)**

46 pxTCB->ulNotifiedValue = ulValue;

47 break;

48

49 /\* 如果被通知任务当前没有通知，则被通知任务的通知值设置为ulValue；

50 在某些场景下替代长度为1的xQueuesend()，但速度更快。 \*/

51 case eSetValueWithoutOverwrite : **(12)**

52 if ( ucOriginalNotifyState != taskNOTIFICATION_RECEIVED ) {

53 pxTCB->ulNotifiedValue = ulValue;

54 } else {

55 /*如果被通知任务还没取走上一个通知，本次发送通知，

56 任务又接收到了一个通知，则这次通知值丢弃，

57 在这种情况下，函数调用失败并返回pdFALSE。*/

58 xReturn = pdFAIL; **(13)**

59 }

60 break;

61

62 /\* 发送通知但不更新通知值，这意味着参数ulValue未使用。 \*/

63 case eNoAction: **(14)**

64 break;

65 }

66

67 traceTASK_NOTIFY();

68

69 /\* 如果被通知任务由于等待任务通知而挂起 \*/

70 if ( ucOriginalNotifyState == taskWAITING_NOTIFICATION ) {**(15)**

71 /\* 唤醒任务，将任务从阻塞列表中移除，添加到就绪列表中 \*/

72 ( void ) uxListRemove( &( pxTCB->xStateListItem ) );

73 prvAddTaskToReadyList( pxTCB );

74

75 // 刚刚唤醒的任务优先级比当前任务高

76 if ( pxTCB->uxPriority > pxCurrentTCB->uxPriority ) {**(16)**

77 //任务切换

78 taskYIELD_IF_USING_PREEMPTION();

79 } else {

80 mtCOVERAGE_TEST_MARKER();

81 }

82 } else {

83 mtCOVERAGE_TEST_MARKER();

84 }

85 }

86 taskEXIT_CRITICAL();

87

88 return xReturn;

89 }

90

91 #endif代码清单20‑2

代码清单20‑2\ **(1)**\ ：被通知的任务句柄，指定通知的任务。

代码清单20‑2\ **(2)**\ ：发送的通知值。

代码清单20‑2\ **(3)**\ ：枚举类型，指明更新通知值的方式。

代码清单20‑2\ **(4)**\ ：任务原本的通知值返回。

代码清单20‑2\ **(5)**\ ：回传任务原本的任务通值，保存在pulPreviousNotificationValue中。

代码清单20‑2\ **(6)**\ ：获取任务通知的状态，看看任务是否在等待通知，方便在发送通知后恢复任务。

代码清单20‑2\ **(7)**\ ：不管该任务的通知状态是怎么样的，现在调用发送通知函数，任务通知状态就要设置为收到任务通知，因为发送通知是肯定能被收到。

代码清单20‑2\ **(8)**\ ：指定更新任务通知的方式。

代码清单20‑2\ **(9)**\ ：通知值与原本的通知值按位或，使用这种方法可以某些场景下代替事件组，执行速度更快。

代码清单20‑2\ **(10)**\ ：被通知任务的通知值增加1，这种发送通知方式，参数ulValue的值未使用，在某些场景可以代替信号量通信，并且执行速度更快。

代码清单20‑2\ **(11)**\ ：将被通知任务的通知值设置为ulValue，无论任务是否还有通知，都覆盖当前任务通知值。这种方法是覆盖写入，使用这种方法，可以在某些场景下代替xQueueoverwrite()函数，执行速度更快。

代码清单20‑2\ **(12)**\ ：如果被通知任务当前没有通知，则被通知任务的通知值设置为ulValue；在某些场景下替代队列长度为1的xQueuesend()，并且执行速度更快。

代码清单20‑2\ **(13)**\ ：如果被通知任务还没取走上一个通知，本次发送通知，任务又接收到了一个通知，则这次通知值将被丢弃，在这种情况下，函数调用失败并返回pdFALSE。

代码清单20‑2\ **(14)**\ ：发送通知但不更新通知值，这意味着参数ulValue未使用。

代码清单20‑2\ **(15)**\ ：如果被通知的任务由于等待任务通知而挂起，系统将唤醒任务，将任务从阻塞列表中移除，添加到就绪列表中。

代码清单20‑2\ **(16)**\ ：如果刚刚唤醒的任务优先级比当前任务高，就进行一次任务切换。

xTaskGenericNotify()函数是一个通用的任务通知发送函数，在任务中发送通知的API函数，如xTaskNotifyGive()、xTaskNotify()、xTaskNotifyAndQuery()，都是以xTaskGenericNotify()为原型的，只不过指定的发生方式不同而已。

xTaskNotifyGive()
'''''''''''''''''

xTaskNotifyGive()是一个宏，宏展开是调用函数xTaskNotify( ( xTaskToNotify ), ( 0 ), eIncrement
)，即向一个任务发送通知，并将对方的任务通知值加1。该函数可以作为二值信号量和计数信号量的一种轻量型的实现，速度更快，在这种情况下对象任务在等待任务通知的时候应该是使用函数 `ulTaskNotifyTake()
<http://www.freertos.org/ulTaskNotifyTake.html>`__ 而不是\ `xTaskNotifyWait()
<http://www.freertos.org/xTaskNotifyWait.html>`__ 。xTaskNotifyGive()不能在中断里面使用，而是使用具有中断保护功能的\ `vTaskNotifyGiveFromISR()
<http://www.freertos.org/vTaskNotifyGiveFromISR.html>`__\ 来代替。该函数的具体说明见表20‑1，应用举例见代码清单20‑3加粗部分。

表20‑1xTaskNotifyGive()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | #d
     - fine xTaskNotifyGive( | xTaskToNotify )  xTaskGenericNotify( ( xTaskToNotify ), ( 0 ), eIncrement, NULL )
     - |

   * - **功能**     |
     - 于在任务中向指定任务发 | 送任务通知，并更新对方的 | 任务通知值（加1操作）。  |
     - |
         |
          |

   * - **参数**     |
     - TaskToNotify            |
     - 收通知的任务句柄，并让 | 其自身的任务通知值加1。  |

   * - **返回值**   | 总
     - 返回pdPASS。         |
     - |


代码清单20‑3xTaskNotifyGive()函数应用举例

1 /\* 函数声明 \*/

2 static void prvTask1( void \*pvParameters );

3 static void prvTask2( void \*pvParameters );

4

5 /*定义任务句柄 \*/

6 static TaskHandle_t xTask1 = NULL, xTask2 = NULL;

7

8 /\* 主函数:创建两个任务，然后开始任务调度 \*/

9 void main( void )

10 {

11 xTaskCreate(prvTask1, "Task1", 200, NULL, tskIDLE_PRIORITY, &xTask1);

12 xTaskCreate(prvTask2, "Task2", 200, NULL, tskIDLE_PRIORITY, &xTask2);

13 vTaskStartScheduler();

14 }

15 /*-----------------------------------------------------------*/

16

17 static void prvTask1( void \*pvParameters )

18 {

19 for ( ;; ) {

20 /\* 向prvTask2()发送一个任务通知，让其退出阻塞状态 \*/

21 xTaskNotifyGive( xTask2 );

22

23 /\* 阻塞在prvTask2()的任务通知上

24 如果没有收到通知，则一直等待*/

25 ulTaskNotifyTake( pdTRUE, portMAX_DELAY );

26 }

27 }

28 /*-----------------------------------------------------------*/

29

30 static void prvTask2( void \*pvParameters )

31 {

32 for ( ;; ) {

33 /\* 阻塞在prvTask1()的任务通知上

34 如果没有收到通知，则一直等待*/

35 ulTaskNotifyTake( pdTRUE, portMAX_DELAY );

36

37 /\* 向prvTask1()发送一个任务通知，让其退出阻塞状态 \*/

38 xTaskNotifyGive( xTask1 );

39 }

40 }

vTaskNotifyGiveFromISR()
''''''''''''''''''''''''

vTaskNotifyGiveFromISR()是vTaskNotifyGive()的中断保护版本。用于在中断中向指定任务发送任务通知，并更新对方的任务通知值（加1操作），在某些场景中可以替代信号量操作，因为这两个通知都是不带有通知值的。该函数的具体说明见表20‑2。

表20‑2vTaskNotifyGiveFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | vo
     - d                     | vTaskNotify GiveFromISR(TaskHandle_t xTaskToNotify,  BaseType_t \*pxH igherPriorityTaskWoken);
     - |

   * - **功能**     |
     - 于在中断中向一个任务发 | 送任务通知，并更新对方的 | 任务通知值（加1操作）。  |
     - |
         |
          |

   * - **参数**     |
     - TaskToNotify            |
     - 收通知的任务句柄，并让 | 其自身的任务通知值加1。  |

   * -
     - p xHigherPriorityTaskWoken
     - \ *pxHigherPriorityTaskWok en在使用之前必须先初始化 | 为pdFALSE。当调用该函数  | 发送一个任务通知时，目标 | 任务接收到通知后将从阻塞 | 态变为就绪态，并且如果其 | 优先级比当前运行的任务的 | 优先级高，那么*pxHigherP |
       riorityTaskWoken会被设置 | 为pdTRUE，然后在中断退出 | 前执行一次上下文切换，去 | 执行刚刚被唤醒的中断优先 | 级较高的任务。pxHigherP  | riorityTaskWoken是一个可 | 选的参数可以设置为NULL。 |

   * - **返回值**   | 无
     - |
     - |


从上面的函数说明我们大概知道vTaskNotifyGiveFromISR()函数作用，每次调用该函数都会增加任务的通知值，任务通过接收函数返回值是否大于零，判断是否获取到了通知，任务通知值初始化为0，（如果与信号量做对比）则对应为信号量无效。当中断调用vTaskNotifyGiveFromISR()
通知函数给任务的时候，任务的通知值增加，使其大于零，使其表示的通知值变为有效，任务获取有效的通知值将会被恢复。那么该函数是怎么实现的呢？下面一起来看看vTaskNotifyGiveFromISR()函数的源码，具体见代码清单20‑4。

代码清单20‑4vTaskNotifyGiveFromISR()源码

1 #if( configUSE_TASK_NOTIFICATIONS == 1 )

2

3 void vTaskNotifyGiveFromISR( TaskHandle_t xTaskToNotify,

4 BaseType_t \*pxHigherPriorityTaskWoken )

5 {

6 TCB_t \* pxTCB;

7 uint8_t ucOriginalNotifyState;

8 UBaseType_t uxSavedInterruptStatus;

9

10 configASSERT( xTaskToNotify );

11

12 portASSERT_IF_INTERRUPT_PRIORITY_INVALID();

13

14 pxTCB = ( TCB_t \* ) xTaskToNotify;

15

16 //进入中断

17 uxSavedInterruptStatus = portSET_INTERRUPT_MASK_FROM_ISR();

18 {

19 //保存任务通知的原始状态，

20 //看看任务是否在等待通知，方便在发送通知后恢复任务

21 ucOriginalNotifyState = pxTCB->ucNotifyState; **(1)**

22

23 /\* 不管状态是怎么样的，反正现在发送通知，任务就收到任务通知 \*/

24 pxTCB->ucNotifyState = taskNOTIFICATION_RECEIVED; **(2)**

25

26 /\* 通知值自加，类似于信号量的释放 \*/

27 ( pxTCB->ulNotifiedValue )++; **(3)**

28

29 traceTASK_NOTIFY_GIVE_FROM_ISR();

30

31 /\* 如果任务在阻塞等待通知 \*/

32 if ( ucOriginalNotifyState == taskWAITING_NOTIFICATION ) {**(4)**

33 //如果任务调度器运行中

34 if ( uxSchedulerSuspended == ( UBaseType_t ) pdFALSE ) {

35 /\* 唤醒任务，将任务从阻塞列表中移除，添加到就绪列表中 \*/

36 ( void ) uxListRemove( &( pxTCB->xStateListItem ) );\ **(5)**

37 prvAddTaskToReadyList( pxTCB );

38 } else {

39 /\* 调度器处于挂起状态，中断依然正常发生，但是不能直接操作就绪列表

40 将任务加入到就绪挂起列表，任务调度恢复后会移动到就绪列表 \*/

41 vListInsertEnd( &( xPendingReadyList ),

42 &( pxTCB->xEventListItem ) );\ **(6)**

43 }

44

45 /\* 如果刚刚唤醒的任务优先级比当前任务高,

46 则设置上下文切换标识,等退出函数后手动切换上下文,

47 或者在系统节拍中断服务程序中自动切换上下文 \*/

48 if ( pxTCB->uxPriority > pxCurrentTCB->uxPriority ) {**(7)**

49 //

50 /\* 设置返回参数，表示需要任务切换，在退出中断前进行任务切换 \*/

51 if ( pxHigherPriorityTaskWoken != NULL ) {

52 \*pxHigherPriorityTaskWoken = pdTRUE; **(8)**

53 } else {

54 /\* 设置自动切换标志 \*/

55 xYieldPending = pdTRUE; **(9)**

56 }

57 } else {

58 mtCOVERAGE_TEST_MARKER();

59 }

60 }

61 }

62 portCLEAR_INTERRUPT_MASK_FROM_ISR( uxSavedInterruptStatus );

63 }

64

65 #endif

代码清单20‑4\ **(1)**\ ：保存任务通知的原始状态，看看任务是否处于等待通知的阻塞态，方便在中断发送通知完成后恢复任务。

代码清单20‑4\ **(2)**\ ：不管状态是怎么样的，反正现在发送通知，任务就收到任务通知。

代码清单20‑4\ **(3)**\ ：通知值自加，类似于信号量的释放操作。

代码清单20‑4\ **(4)**\ ：如果任务在阻塞等待通知，并且系统调度器处于运行状态。

代码清单20‑4\ **(5)**\ ：唤醒任务，将任务从阻塞列表中移除，添加到就绪列表中。

代码清单20‑4\ **(6)**\ ：调度器处于挂起状态，中断依然正常发生，但是不能直接操作就绪列表，将任务加入到就绪挂起列表，任务调度恢复后会移动到就绪列表中。

代码清单20‑4\ **(7)**\ ：如果刚刚唤醒的任务优先级比当前任务高，则设置上下文切换标识，等退出函数后手动切换上下文，或者在系统节拍中断服务程序中自动切换上下文

代码清单20‑4\ **(8)**\ ：设置返回参数，表示需要任务切换，在退出中断前进行任务切换。

代码清单20‑4\ **(9)**\ ：否则就设置自动切换标志。

代码清单20‑5vTaskNotifyGiveFromISR()函数应用举例

1 static TaskHandle_t xTaskToNotify = NULL;

2

3 /\* 外设驱动的数据传输函数 \*/

4 void StartTransmission( uint8_t \*pcData, size_t xDataLength )

5 {

6 /\* 在这个时候，xTaskToNotify应为NULL，因为发送并没有进行。

7 如果有必要，对外设的访问可以用互斥量来保护*/

8 configASSERT( xTaskToNotify == NULL );

9

10 /\* 获取调用函数StartTransmission()的任务的句柄 \*/

11 xTaskToNotify = xTaskGetCurrentTaskHandle();

12

13 /\* 开始传输，当数据传输完成时产生一个中断 \*/

14 vStartTransmit( pcData, xDatalength );

15 }

16 /*-----------------------------------------------------------*/

17 /\* 数据传输完成中断 \*/

18 void vTransmitEndISR( void )

19 {

20 BaseType_t xHigherPriorityTaskWoken = pdFALSE;

21

22 /\* 这个时候不应该为NULL，因为数据传输已经开始 \*/

23 configASSERT( xTaskToNotify != NULL );

24

25 /\* 通知任务传输已经完成 \*/

26 vTaskNotifyGiveFromISR( xTaskToNotify, &xHigherPriorityTaskWoken );

27

28 /\* 传输已经完成，所以没有任务需要通知 \*/

29 xTaskToNotify = NULL;

30

31 /\* 如果为pdTRUE，则进行一次上下文切换 \*/

32 portYIELD_FROM_ISR( xHigherPriorityTaskWoken );

33 }

34 /*-----------------------------------------------------------*/

35 /\* 任务：启动数据传输，然后进入阻塞态，直到数据传输完成 \*/

36 void vAFunctionCalledFromATask( uint8_t ucDataToTransmit,

37 size_t xDataLength )

38 {

39 uint32_t ulNotificationValue;

40 const TickType_t xMaxBlockTime = pdMS_TO_TICKS( 200 );

41

42 /\* 调用上面的函数StartTransmission()启动传输 \*/

43 StartTransmission( ucDataToTransmit, xDataLength );

44

45 /\* 等待传输完成 \*/

46 ulNotificationValue = ulTaskNotifyTake( pdFALSE, xMaxBlockTime );

47

48 /\* 当传输完成时，会产生一个中断

49 在中断服务函数中调用vTaskNotifyGiveFromISR()向启动数据

50 传输的任务发送一个任务通知，并将对象任务的任务通知值加1

51 任务通知值在任务创建的时候是初始化为0的，当接收到任务后就变成1 \*/

52 if ( ulNotificationValue == 1 ) {

53 /\* 传输按预期完成 \*/

54 } else {

55 /\* 调用函数ulTaskNotifyTake()超时 \*/

56 }

57 }

xTaskNotify()
'''''''''''''

FreeRTOS每个任务都有一个32位的变量用于实现任务通知，在任务创建的时候初始化为0。这个32位的通知值在任务控制块TCB里面定义，具体见代码清单20‑6。xTaskNotify()用于在任务中直接向另外一个任务发送一个事件，接收到该任务通知的任务有可能解锁。如果你想使用任务通知来实现二值信号量
和计数信号量，那么应该使用更加简单的函数\ `xTaskNotifyGive()
<http://www.freertos.org/xTaskNotifyGive.html>`__ ，而不是使用xTaskNotify()，xTaskNotify()函数在发送任务通知的时候会指定一个通知值，并且用户可以指定通知值发送的方式。

注意：该函数不能在中断里面使用，而是使用具体中断保护功能的版本函数\ `xTaskNotifyFromISR() <http://www.freertos.org/xTaskNotifyFromISR.html>`__\
。xTaskNotify()函数的具体说明见表20‑3，应用举例见代码清单20‑6。

代码清单20‑6任务通知在任务控制块中的定义

1 #if( configUSE_TASK_NOTIFICATIONS == 1 )

2 volatileuint32_t ulNotifiedValue;

3 volatileuint8_t ucNotifyState;

4 #endif

表20‑3xTaskNotify()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t xTaskNotify(  | TaskHandle_t xTaskToNotify,  uint32_t ulValue,  eNotifyAction eAction );
     - |

   * - **功能**     |
     - | 指定的任务发送一个任务通 | 知，带有通知值并且用户可 | 以指定通知值的发送方式。 |
     - |

   * - **参数**     |
     - TaskToNotify            |
     - 要接收通知的任务句柄。 |

   * -
     - ulValue
     - 用于更新接收任务通       | 知的任务通知值，具体如何 | 更新由形参eAction决定。  |

   * -
     - eAction
     - 任务通知值               | 更新方式，具体见表20‑4。 |

   * - **返回值**   | 参
     - eAction为            | eSetValueWithoutOverwri te时，如果被通知任务还没 | 取走上一个通知，又接收到 | 了一个通知，则这次通知值 | 未能更新并返回pdFALSE，  | 而其他情况均返回pdPASS。 |
     - |


表20‑4任务通知值的状态

.. list-table::
   :widths: 50 50
   :header-rows: 0


   * - eAction取值               |
     - 义                                    |

   * - eNoAction
     - 对象任务接收任务通知，但是任务自身的    | 任务通知值不更新，即形参ulValue没有用。 |

   * - eSetBits
     - 对象任务接收任务通知，同                | 时任务自身的任务通知值与ulValue按位或。 | 如果ulValue设置为0x01，那么任务的通知值 | 的位0将被置为1。同样的如果ulValue设置为 | 0x04，那么任务的通知值的位2将被置为1。  |
       在这种方式下，任务通知可以看成是        | 事件标志的一种轻量型的实现，速度更快。  |

   * - eIncrement
     - 对象任                                  | 务接收任务通知，任务自身的任务通知值加  | 1，即形参ulValue没有用。这个时候调用xTa | skNotify()等同于调用xTaskNotifyGive()。 |

   * - eSetValueWithOverwrite
     - 对象任务接收任务通知，且任务自身        | 的任务通知值会无条件的被设置为ulValue。 |  在这种方式下，任务通知                  | 可以看成是函数\ `xQueueOverwrite() <ht  |
       tp://www.freertos.org/xQueueOverwrite.h tml>`__\ 的一种轻量型的实现，速度更快。 |

   * - eSetValueWithoutOverwrite
     - 对象任务接收任务通知，且对象任务没有    | 通知值，那么通知值就会被设置为ulValue。 |  对象任务接收任务通知，但是上一          | 次接收到的通知值并没有取走，那么本次的  | 通知值将不会更新，同时函数返回pdFALSE。 |  在这种方式下，任务通知可以看成
       | 是函数\ `xQueueSend() <http://www.free  | rtos.org/a00117.html>`__ 应用在队列深度 | 为1的队列上的一种轻量型实现，速度更快。 |


代码清单20‑7xTaskNotify()函数应用举例

1 /\* 设置任务xTask1Handle的任务通知值的位8为1*/

2 xTaskNotify( xTask1Handle, ( 1UL << 8UL ), eSetBits );

3

4 /\* 向任务xTask2Handle发送一个任务通知

5 有可能会解除该任务的阻塞状态，但是并不会更新该任务自身的任务通知值 \*/

6 xTaskNotify( xTask2Handle, 0, eNoAction );

7

8

9 /\* 向任务xTask3Handle发送一个任务通知

10 并把该任务自身的任务通知值更新为0x50

11 即使该任务的上一次的任务通知都没有读取的情况下

12 即覆盖写 \*/

13 xTaskNotify( xTask3Handle, 0x50, eSetValueWithOverwrite );

14

15 /\* 向任务xTask4Handle发送一个任务通知

16 并把该任务自身的任务通知值更新为0xfff

17 但是并不会覆盖该任务之前接收到的任务通知值*/

18 if(xTaskNotify(xTask4Handle,0xfff,eSetValueWithoutOverwrite)==pdPASS )

19 {

20/\* 任务xTask4Handle的任务通知值已经更新 \*/

21} else

22{

23/\* 任务xTask4Handle的任务通知值没有更新

24即上一次的通知值还没有被取走*/

25}

xTaskNotifyFromISR()
''''''''''''''''''''

xTaskNotifyFromISR()是xTaskNotify()的中断保护版本，真正起作用的函数是中断发送任务通知通用函数xTaskGenericNotifyFromISR()，而xTaskNotifyFromISR()是一个宏定义，具体见代码清单20‑8，用于在中断中向指定的任务发送一个任务通
知，该任务通知是带有通知值并且用户可以指定通知的发送方式，不返回上一个任务在的通知值。函数的具体说明见表20‑5。xTaskGenericNotifyFromISR()的源码具体见代码清单20‑9。

代码清单20‑8 xTaskNotifyFromISR()函数原型

1 #define xTaskNotifyFromISR( xTaskToNotify, \\

2 ulValue, \\

3 eAction, \\

4 pxHigherPriorityTaskWoken ) \\

5 xTaskGenericNotifyFromISR( ( xTaskToNotify ), \\

6 ( ulValue ), \\

7 ( eAction ), \\

8 NULL, \\

9 ( pxHigherPriorityTaskWoken ) )

表20‑5xTaskNotifyFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xTaskNotifyFromISR( TaskHandle_t xTaskToNotify,  uint32_t ulValue,  eNotifyAction eAction,  BaseType_t \*p
       xHigherPriorityTaskWoken );
     - |

   * - **功能**     |
     - 中断中向指定           | 的任务发送一个任务通知。 |
     - |

   * - **参数**     |
     - TaskToNotify            |
     - 定接收通知的任务句柄。 |

   * -
     - ulValue
     - 用于更新接收任务通       | 知的任务通知值，具体如何 | 更新由形参eAction决定。  |

   * -
     - eAction
     - 任务通知                 | 值的状态，具体见表20‑4。 |

   * -
     - p xHigherPriorityTaskWoken
     - \ *pxHigherPriorityTaskWok en在使用之前必须先初始化 | 为pdFALSE。当调用该函数  | 发送一个任务通知时，目标 | 任务接收到通知后将从阻塞 | 态变为就绪态，并且如果其 | 优先级比当前运行的任务的 | 优先级高，那么*pxHigherP |
       riorityTaskWoken会被设置 | 为pdTRUE，然后在中断退出 | 前执行一次上下文切换，去 | 执行刚刚被唤醒的中断优先 | 级较高的任务。pxHigherP  | riorityTaskWoken是一个可 | 选的参数可以设置为NULL。 |

   * - **返回值**   | 参
     - eActio               | n为eSetValueWithoutOverw | rite时，如果被通知任务还 | 没取走上一个通知，又接收 | 到了一个通知，则这次通知 | 值未能更新并返回pdFALSE  | ，其他情况均返回pdPASS。 |
     - |
           |


中断中发送任务通知通用函数xTaskGenericNotifyFromISR()
''''''''''''''''''''''''''''''''''''''''

xTaskGenericNotifyFromISR()是一个在中断中发送任务通知的通用函数，xTaskNotifyFromISR()、xTaskNotifyAndQueryFromISR()等函数都是以其为基础，采用宏定义的方式实现。xTaskGenericNotifyFromISR()的源码具体见
代码清单20‑9。

代码清单20‑9xTaskGenericNotifyFromISR()源码

1 #if( configUSE_TASK_NOTIFICATIONS == 1 )

2

3 BaseType_t xTaskGenericNotifyFromISR( TaskHandle_t xTaskToNotify,\ **(1)**

4 uint32_t ulValue, **(2)**

5 eNotifyAction eAction, **(3)**

6 uint32_t \*pulPreviousNotificationValue,\ **(4)**

7 BaseType_t \*pxHigherPriorityTaskWoken )\ **(5)**

8 {

9 TCB_t \* pxTCB;

10 uint8_t ucOriginalNotifyState;

11 BaseType_t xReturn = pdPASS;

12 UBaseType_t uxSavedInterruptStatus;

13

14 configASSERT( xTaskToNotify );

15

16 portASSERT_IF_INTERRUPT_PRIORITY_INVALID();

17

18 pxTCB = ( TCB_t \* ) xTaskToNotify;

19

20 /\* 进入中断临界区 \*/

21 uxSavedInterruptStatus = portSET_INTERRUPT_MASK_FROM_ISR(); **(6)**

22 {

23 if ( pulPreviousNotificationValue != NULL ) {

24 /*回传未被更新的任务通知值*/

25 \*pulPreviousNotificationValue = pxTCB->ulNotifiedValue;\ **(7)**

26 }

27

28 //保存任务通知的原始状态，

29 //看看任务是否在等待通知，方便在发送通知后恢复任务

30 ucOriginalNotifyState = pxTCB->ucNotifyState; **(8)**

31

32 /\* 不管状态是怎么样的，反正现在发送通知，任务就收到任务通知 \*/

33 pxTCB->ucNotifyState = taskNOTIFICATION_RECEIVED; **(9)**

34

35 /\* 指定更新任务通知的方式 \*/

36 switch ( eAction ) { **(10)**

37 /*通知值按位或上ulValue。

38 使用这种方法可以某些场景下代替事件组，但执行速度更快。*/

39 case eSetBits : **(11)**

40 pxTCB->ulNotifiedValue \|= ulValue;

41 break;

42

43 /\* 被通知任务的通知值增加1，这种发送通知方式，参数ulValue未使用

44 在某些场景下可以代替信号量，执行速度更快 \*/

45 case eIncrement: **(12)**

46 ( pxTCB->ulNotifiedValue )++;

47 break;

48

49 /\* 将被通知任务的通知值设置为ulValue。无论任务是否还有通知，

50 都覆盖当前任务通知值。使用这种方法，

51 可以在某些场景下代替xQueueoverwrite()函数，但执行速度更快。 \*/

52 case eSetValueWithOverwrite: **(13)**

53 pxTCB->ulNotifiedValue = ulValue;

54 break;

55

56 //采用不覆盖发送任务通知的方式

57 case eSetValueWithoutOverwrite : **(14)**

58 /\* 如果被通知任务当前没有通知，则被通知任务的通知值设置为ulValue；

59 在某些场景下替代长度为1的xQueuesend()，但速度更快。 \*/

60 if ( ucOriginalNotifyState != taskNOTIFICATION_RECEIVED ) {

61 pxTCB->ulNotifiedValue = ulValue;

62 } else {

63 /*如果被通知任务还没取走上一个通知，本次发送通知，

64 任务又接收到了一个通知，则这次通知值丢弃，

65 在这种情况下，函数调用失败并返回pdFALSE。*/

66 xReturn = pdFAIL; **(15)**

67 }

68 break;

69

70 case eNoAction :

71 /\* 退出 \*/

72 break;

73 }

74

75 traceTASK_NOTIFY_FROM_ISR();

76

77 /\* 如果任务在阻塞等待通知*/

78 if ( ucOriginalNotifyState == taskWAITING_NOTIFICATION ) {**(16)**

79 //如果任务调度器运行中，表示可用操作就绪级列表

80 if ( uxSchedulerSuspended == ( UBaseType_t ) pdFALSE ) {

81 /\* 唤醒任务，将任务从阻塞列表中移除，添加到就绪列表中 \*/

82 ( void ) uxListRemove( &( pxTCB->xStateListItem ) );

83 prvAddTaskToReadyList( pxTCB ); **(17)**

84 } else {

85 /\* 调度器处于挂起状态，中断依然正常发生，但是不能直接操作就绪列表

86 将任务加入到就绪挂起列表，任务调度恢复后会移动到就绪列表 \*/

87 vListInsertEnd( &( xPendingReadyList ),

88 &( pxTCB->xEventListItem ) ); **(18)**

89 }

90 /\* 如果刚刚唤醒的任务优先级比当前任务高,

91 则设置上下文切换标识,等退出函数后手动切换上下文,

92 或者自动切换上下文 \*/

93 if ( pxTCB->uxPriority > pxCurrentTCB->uxPriority ) {**(19)**

94

95 if ( pxHigherPriorityTaskWoken != NULL ) {

96 /\* 设置返回参数，表示需要任务切换，在退出中断前进行任务切换 \*/

97 \*pxHigherPriorityTaskWoken = pdTRUE; **(20)**

98 } else {

99 /*设置自动切换标志，等高优先级任务释放CPU使用权 \*/

100 xYieldPending = pdTRUE; **(21)**

101 }

102 } else {

103 mtCOVERAGE_TEST_MARKER();

104 }

105 }

106 }

107 /\* 离开中断临界区 \*/

108 portCLEAR_INTERRUPT_MASK_FROM_ISR( uxSavedInterruptStatus );\ **(22)**

109

110 return xReturn;

111 }

112

113 #endif

代码清单20‑9\ **(1)**\ ：指定接收通知的任务句柄。

代码清单20‑9\ **(2)**\ ：用于更新接收任务通知值，具体如何更新由形参eAction决定。

代码清单20‑9\ **(3)**\ ：任务通知值更新方式。

代码清单20‑9\ **(4)**\ ：用于保存上一个任务通知值。

代码清单20‑9\ **(5)**\ ：*pxHigherPriorityTaskWoken在使用之前必须先初始化为pdFALSE。当调用该函数发送一个任务通知时，目标任务接收到通知后将从阻塞态变为就绪态，并且如果其优先级比当前运行的任务的优先级高，那么*pxHigherPriorityTaskWo
ken会被设置为pdTRUE，然后在中断退出前执行一次上下文切换，去执行刚刚被唤醒的中断优先级较高的任务。pxHigherPriorityTaskWoken是一个可选的参数可以设置为NULL。

代码清单20‑9\ **(6)**\ ：进入中断临界区。

代码清单20‑9\ **(7)**\ ：如果pulPreviousNotificationValue参数不为空，就需要返回上一次的任务通知值。

代码清单20‑9\ **(8)**\ ：保存任务通知的原始状态，看看任务是否在等待通知，方便在发送通知后恢复任务。

代码清单20‑9\ **(9)**\ ：不管当前任务通知状态是怎么样的，现在调用发送通知函数。任务通知肯定是发送到指定任务，那么任务通知的状态就设置为收到任务通知。

代码清单20‑9\ **(10)**\ ：指定更新任务通知的方式。

代码清单20‑9\ **(11)**\ ：通知值与原本的通知值按位或，使用这种方法可以某些场景下代替事件组，执行速度更快。

代码清单20‑9\ **(12)**\ ：被通知任务的通知值增加1，这种发送通知方式，参数ulValue的值未使用，在某些场景可以代替信号量通信，并且执行速度更快。

代码清单20‑9\ **(13)**\ ：将被通知任务的通知值设置为ulValue，无论任务是否还有通知，都覆盖当前任务通知值。这种方法是覆盖写入，使用这种方法，可以在某些场景下代替xQueueoverwrite()函数，执行速度更快。

代码清单20‑9\ **(14)**\ ：采用不覆盖发送通知方式，如果被通知任务当前没有通知，则被通知任务的通知值设置为ulValue；在某些场景下替代队列长度为1的xQueuesend()，并且执行速度更快。

代码清单20‑9\ **(15)**\ ：如果被通知任务还没取走上一个通知，本次发送通知，任务又接收到了一个通知，则这次通知值将被丢弃，在这种情况下，函数调用失败并返回pdFALSE。

代码清单20‑9\ **(16)**\ ：如果任务在阻塞等待通知。

代码清单20‑9\ **(17)**\ ：如果任务调度器在运行中，表示可用操作就绪级列表。那么系统将唤醒任务，将任务从阻塞列表中移除，添加到就绪列表中

代码清单20‑9\ **(18)**\ ：如果调度器处于挂起状态，中断依然正常发生，但是不能直接操作就绪列表，系统会将任务加入到就绪挂起列表，任务调度恢复后会将在该列表的任务移动到就绪列表中。

代码清单20‑9\ **(19)**\ ：如果刚刚唤醒的任务优先级比当前任务高，则设置上下文切换标识,等退出函数后手动切换上下文，或者按照任务优先级自动切换上下文。

代码清单20‑9\ **(20)**\ ：设置返回参数，表示需要任务切换，在退出中断前进行任务切换。

代码清单20‑9\ **(21)**\ ：设置自动切换标志，等高优先级任务释放CPU使用权。

代码清单20‑9\ **(22)**\ ：离开中断临界区

xTaskNotifyFromISR()的使用很简单的，具体见代码清单20‑10加粗部分。

代码清单20‑10xTaskNotifyFromISR()使用实例

1 /\* 中断：向一个任务发送任务通知，并根据不同的中断将目标任务的

2 任务通知值的相应位置1 \*/

3 void vANInterruptHandler( void )

4 {

5 BaseType_t xHigherPriorityTaskWoken;

6 uint32_t ulStatusRegister;

7

**8 /\* 读取中断状态寄存器，判断到来的是哪个中断**

**9 这里假设了Rx、Tx和buffer overrun 三个中断 \*/**

**10 ulStatusRegister = ulReadPeripheralInterruptStatus();**

11

12 /\* 清除中断标志位 \*/

13 vClearPeripheralInterruptStatus( ulStatusRegister );

14

15 /\* xHigherPriorityTaskWoken 在使用之前必须初始化为pdFALSE

16 如果调用函数xTaskNotifyFromISR()解锁了解锁了接收该通知的任务

17 而且该任务的优先级比当前运行的任务的优先级高，那么

18 xHigherPriorityTaskWoken就会自动的被设置为pdTRUE*/

19 xHigherPriorityTaskWoken = pdFALSE;

20

21 /\* 向任务xHandlingTask发送任务通知，并将其任务通知值

22 与ulStatusRegister的值相或，这样可以不改变任务通知其他位的值*/

**23 xTaskNotifyFromISR( xHandlingTask,**

**24 ulStatusRegister,**

**25 eSetBits,**

**26 &xHigherPriorityTaskWoken );**

27

28 /\* 如果xHigherPriorityTaskWoken的值为pdRTUE

29 则执行一次上下文切换*/

30 portYIELD_FROM_ISR( xHigherPriorityTaskWoken );

31 }

32 /\* ----------------------------------------------------------- \*/

33

34

35 /\* 任务：等待任务通知，然后处理相关的事情 \*/

36 void vHandlingTask( void \*pvParameters )

37 {

38 uint32_t ulInterruptStatus;

39

40 for ( ;; ) {

41 /\* 等待任务通知，无限期阻塞（没有超时，所以没有必要检查函数返回值）*/

42 xTaskNotifyWait( 0x00, /\* 在进入的时候不清除通知值的任何位 \*/

43 ULONG_MAX, /\* 在退出的时候复位通知值为0 \*/

44 &ulNotifiedValue, /\* 任务通知值传递到变量

45 ulNotifiedValue中*/

46 portMAX_DELAY ); /\* 无限期等待 \*/

47

48 /\* 根据任务通知值里面的各个位的值处理事情 \*/

49 if ( ( ulInterruptStatus & 0x01 ) != 0x00 ) {

50 /\* Rx中断 \*/

51 prvProcessRxInterrupt();

52 }

53

54 if ( ( ulInterruptStatus & 0x02 ) != 0x00 ) {

55 /\* Tx中断 \*/

56 prvProcessTxInterrupt();

57 }

58

59 if ( ( ulInterruptStatus & 0x04 ) != 0x00 ) {

60 /\* 缓冲区溢出中断 \*/

61 prvClearBufferOverrun();

62 }

63 }

64 }

xTaskNotifyAndQuery()
'''''''''''''''''''''

xTaskNotifyAndQuery()与xTaskNotify()很像，都是调用通用的任务通知发送函数xTaskGenericNotify()来实现通知的发送，不同的是多了一个附加的参数pulPreviousNotifyValue用于回传接收任务的上一个通知值，函数原型具体见代码清单20‑11。
xTaskNotifyAndQuery()函数不能用在中断中，而是必须使用带中断保护功能的xTaskNotifyAndQuery()FromISR来代替。该函数的具体说明见表20‑6，应用举例见代码清单20‑12加粗部分。

代码清单20‑11xTaskNotifyAndQuery()函数原型

1 #define xTaskNotifyAndQuery( xTaskToNotify, \\

2 ulValue, \\

3 eAction, \\

4 pulPreviousNotifyValue ) \\

5 xTaskGenericNotify( ( xTaskToNotify ), \\

6 ( ulValue ), \\

7 ( eAction ), \\

8 ( pulPreviousNotifyValue ) )

表20‑6xTaskNotifyAndQuery()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xTaskNotifyAndQuery( TaskHandle_t xTaskToNotify,  uint32_t ulValue,  eNotifyAction eAction,  uint32_t \*pulPreviousNotifyValue
       );
     - |

   * - **功能**     |
     - 指定的任务             | 发送一个任务通知，并返回 | 对象任务的上一个通知值。 |
     - |

   * - **参数**     |
     - TaskToNotify            |
     - 要接收通知的任务句柄。 |

   * -
     - ulValue
     - 用于更新接收任务通       | 知的任务通知值，具体如何 | 更新由形参eAction决定。  |

   * -
     - eAction
     - 任务通知值               | 更新方式，具体见表20‑4。 |

   * -
     - pulPreviousNotifyValue
     - 对象任务的上一个任       | 务通知值，如果为NULL，则 | 不需要回传，这个时候就等 | 价于函数xTaskNotify()。  |

   * - **返回值**   | 参
     - eActio               | n为eSetValueWithoutOverw | rite时，如果被通知任务还 | 没取走上一个通知，又接收 | 到了一个通知，则这次通知 | 值未能更新并返回pdFALSE  | ，其他情况均返回pdPASS。 |
     - |
           |


代码清单20‑12xTaskNotifyAndQuery()函数应用举例

1 uint32_t ulPreviousValue;

2

3 /\* 设置对象任务xTask1Handle的任务通知值的位8为1

4 在更新位8的值之前把任务通知值回传存储在变量ulPreviousValue中*/

**5 xTaskNotifyAndQuery( xTask1Handle, ( 1UL << 8UL ), eSetBits, &ulPreviousValue );**

6

7

8 /\* 向对象任务xTask2Handle发送一个任务通知，有可能解除对象任务的阻塞状态

9 但是不更新对象任务的通知值，并将对象任务的通知值存储在变量ulPreviousValue中 \*/

**10 xTaskNotifyAndQuery( xTask2Handle, 0, eNoAction, &ulPreviousValue );**

11

12 /\* 覆盖式设置对象任务的任务通知值为0x50

13 且对象任务的任务通知值不用回传，则最后一个形参设置为NULL \*/

**14 xTaskNotifyAndQuery( xTask3Handle, 0x50, eSetValueWithOverwrite, NULL );**

15

16 /\* 设置对象任务的任务通知值为0xfff，但是并不会覆盖对象任务通过

17 xTaskNotifyWait()和ulTaskNotifyTake()这两个函数获取到的已经存在

18 的任务通知值。对象任务的前一个任务通知值存储在变量ulPreviousValue中*/

**19 if ( xTaskNotifyAndQuery( xTask4Handle,**

**20 0xfff,**

**21 eSetValueWithoutOverwrite,**

**22 &ulPreviousValue ) == pdPASS )**

23 {

24 /\* 任务通知值已经更新 \*/

25 } else

26 {

27 /\* 任务通知值没有更新 \*/

28 }

xTaskNotifyAndQueryFromISR()
''''''''''''''''''''''''''''

xTaskNotifyAndQueryFromISR()是xTaskNotifyAndQuery
()的中断版本，用于向指定的任务发送一个任务通知，并返回对象任务的上一个通知值，该函数也是一个宏定义，真正实现发送通知的是xTaskGenericNotifyFromISR()。xTaskNotifyAndQueryFromISR()函数说明见表20‑7，使用实例具体见代码清单20‑13。

表20‑7xTaskNotifyAndQueryFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xTaskNotifyAndQ ueryFromISR(TaskHandle_t xTaskToNotify,  uint32_t ulValue,  eNotifyAction eAction,  uint32_t \
       *pulPreviousNotifyValue,  BaseType_t \*p xHigherPriorityTaskWoken );
     - |

   * - **功能**     |
     - 中断中向指定的任务     | 发送一个任务通知，并返回 | 对象任务的上一个通知值。 |
     - |
       |
       |

   * - **参数**     |
     - TaskToNotify            |
     - 要接收通知的任务句柄。 |

   * -
     - ulValue
     - 用于更新接收任务通       | 知的任务通知值，具体如何 | 更新由形参eAction决定。  |

   * -
     - eAction
     - 任务通知                 | 值的状态，具体见表20‑4。 |

   * -
     - pulPreviousNotifyValue
     - 对象任                   | 务的上一个任务通知值。如 | 果为NULL，则不需要回传。 |

   * -
     - p xHigherPriorityTaskWoken
     - \ *pxHigherPriorityTaskWok en在使用之前必须先初始化 | 为pdFALSE。当调用该函数  | 发送一个任务通知时，目标 | 任务接收到通知后将从阻塞 | 态变为就绪态，并且如果其 | 优先级比当前运行的任务的 | 优先级高，那么*pxHigherP |
       riorityTaskWoken会被设置 | 为pdTRUE，然后在中断退出 | 前执行一次上下文切换，去 | 执行刚刚被唤醒的中断优先 | 级较高的任务。pxHigherP  | riorityTaskWoken是一个可 | 选的参数可以设置为NULL。 |

   * - **返回值**   | 参
     - eActio               | n为eSetValueWithoutOverw | rite时，如果被通知任务还 | 没取走上一个通知，又接收 | 到了一个通知，则这次通知 | 值未能更新并返回pdFALSE  | ，其他情况均返回pdPASS。 |
     - |
           |


代码清单20‑13xTaskNotifyAndQueryFromISR()函数应用举例

1 void vAnISR( void )

2 {

3 /\* xHigherPriorityTaskWoken在使用之前必须设置为pdFALSE \*/

4 BaseType_t xHigherPriorityTaskWoken = pdFALSE.

5 uint32_t ulPreviousValue;

6

7 /\* 设置目标任务xTask1Handle的任务通知值的位8为1

8 在任务通知值的位8被更新之前把上一次的值存储在变量ulPreviousValue中*/

**9 xTaskNotifyAndQueryFromISR( xTask1Handle,**

**10 ( 1UL << 8UL ),**

**11 eSetBits,**

**12 &ulPreviousValue,**

**13 &xHigherPriorityTaskWoken );**

14

15 /\* 如果任务xTask1Handle阻塞在任务通知上，那么现在已经被解锁进入就绪态

16 如果其优先级比当前正在运行的任务的优先级高，则xHigherPriorityTaskWoken

17 会被设置为pdRTUE，然后在中断退出前执行一次上下文切换，在中断退出后则去

18 执行这个被唤醒的高优先级的任务 \*/

19 portYIELD_FROM_ISR( xHigherPriorityTaskWoken );

20 }

获取任务通知函数
^^^^^^^^

既然FreeRTOS中发送任务的函数有那么多个，那么任务怎么获取到通知呢？我们说了，任务通知在某些场景可以替代信号量、消息队列、事件等。获取任务通知函数只能用在任务中，没有带中断保护版本，因此只有两个API函数：ulTaskNotifyTake()和xTaskNotifyWait
()。前者是为代替二值信号量和计数信号量而专门设计的，它和发送通知API函数xTaskNotifyGive()、vTaskNotifyGiveFromISR()配合使用；后者是全功能版的等待通知，可以根据不同的参数实现轻量级二值信号量、计数信号量、事件组和长度为1的队列。

所有的获取任务通知API函数都带有指定阻塞超时时间参数，当任务因为等待通知而进入阻塞时，用来指定任务的阻塞时间，这些超时机制与FreeRTOS的消息队列、信号量、事件等的超时机制一致。

ulTaskNotifyTake()
''''''''''''''''''

ulTaskNotifyTake()作为二值信号量和计数信号量的一种轻量级实现，速度更快。如果FreeRTOS中使用函数xSemaphoreTake() 来获取信号量，这个时候则可以试试使用函数ulTaskNotifyTake()来代替。

对于这个函数，任务通知值为0，对应信号量无效，如果任务设置了阻塞等待，任务被阻塞挂起。当其他任务或中断发送了通知值使其不为0后，通知变为有效，等待通知的任务将获取到通知，并且在退出时候根据用户传递的第一个参数xClearCountOnExit选择清零通知值或者执行减一操作。

xTaskNotifyTake()在退出的时候处理任务的通知值的时候有两种方法，一种是在函数退出时将通知值清零，这种方法适用于实现二值信号量；另外一种是在函数退出时将通知值减1，这种方法适用于实现计数信号量。

当一个任务使用其自身的任务通知值作为二值信号量或者计数信号量时，其他任务应该使用函数xTaskNotifyGive()或者xTaskNotify( ( xTaskToNotify ), ( 0 ), eIncrement
)来向其发送信号量。如果是在中断中，则应该使用他们的中断版本函数。该函数的具体说明见表20‑8。

表20‑8ulTaskNotifyTake()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | ui
     - t32_t                 | ulTaskNotifyTake( BaseType_t xClearCountOnExit,  TickType_t xTicksToWait );
     - |

   * - **功能**     |
     - 于获取一个任务         | 通知，获取二值信号量、计 | 数信号量类型的任务通知。 |
     - |

   * - **参数**     |
     - ClearCountOnExit        |
     - 置为pdFA               | LSE时，函数xTaskNotifyTa | ke()退出前，将任务的通知 | 值减1，可以用来实现计数  | 信号量；设置为pdTRUE时， | 函数xTaskNotifyTake()退  | 出前，将任务通知值清零， | 可以用来实现二值信号量。 |

   * -
     - xTicksToWait
     - 超                       | 时时间，单位为系统节拍周 | 期。宏pdMS_TO_TICKS用于  | 将毫秒转化为系统节拍数。 |

   * - **返回值**   | 返
     - 任务的当前通知       | 值，在其减1或者清0之前。 |
     - |
        |


下面一起来看看ulTaskNotifyTake()源码的实现过程，其实也是很简单的，具体见代码清单20‑14。

代码清单20‑14ulTaskNotifyTake()源码

1 #if( configUSE_TASK_NOTIFICATIONS == 1 )

2

3 uint32_t ulTaskNotifyTake( BaseType_t xClearCountOnExit,

4 TickType_t xTicksToWait )

5 {

6 uint32_t ulReturn;

7

8 taskENTER_CRITICAL(); //进入中断临界区

9 {

10 // 如果通知值为 0 ，阻塞任务

11 // 默认初始化通知值为 0，说明没有未读通知

12 if ( pxCurrentTCB->ulNotifiedValue == 0UL ) { **(1)**

13 /\* 标记任务状态：等待消息通知 \*/

14 pxCurrentTCB->ucNotifyState = taskWAITING_NOTIFICATION;

15

16 //用户指定超时时间了，那就进入等待状态

17 if ( xTicksToWait > ( TickType_t ) 0 ) { **(2)**

18 //根据用户指定超时时间将任务添加到延时列表

19 prvAddCurrentTaskToDelayedList( xTicksToWait, pdTRUE );

20 traceTASK_NOTIFY_TAKE_BLOCK();

21

22 // 切换任务

23 portYIELD_WITHIN_API();

24 } else {

25 mtCOVERAGE_TEST_MARKER();

26 }

27 } else {

28 mtCOVERAGE_TEST_MARKER();

29 }

30 }

31 taskEXIT_CRITICAL();

32 // 到这里说明其他任务或中断向这个任务发送了通知,或者任务阻塞超时,现在继续处理

33 taskENTER_CRITICAL(); **(3)**

34 {

35 // 获取任务通知值

36 traceTASK_NOTIFY_TAKE();

37 ulReturn = pxCurrentTCB->ulNotifiedValue;

38

39 // 看看任务通知是否有效，有效则返回

40 if ( ulReturn != 0UL ) { **(4)**

41 //是否需要清除通知

42 if ( xClearCountOnExit != pdFALSE ) { **(5)**

43 pxCurrentTCB->ulNotifiedValue = 0UL;

44 } else {

45 // 不清除，就减一

46 pxCurrentTCB->ulNotifiedValue = ulReturn - 1; **(6)**

47 }

48 } else {

49 mtCOVERAGE_TEST_MARKER();

50 }

51

52 //恢复任务通知状态变量

53 pxCurrentTCB->ucNotifyState = taskNOT_WAITING_NOTIFICATION;\ **(7)**

54 }

55 taskEXIT_CRITICAL();

56

57 return ulReturn;

58 }

59

60 #endif

代码清单20‑14\ **(1)**\ ：进入临界区，先看看任务通知值是否有效，有效才能获取，无效则根据指定超时时间等待，标记一下任务状态，表示任务在等待通知。任务通知在任务初始化的时候是默认为无效的。

代码清单20‑14\ **(2)**\ ：用户指定超时时间了，那就进入等待状态，根据用户指定超时时间将任务添加到延时列表，然后切换任务，触发PendSV中断，等到退出临界区时立即执行任务切换。

代码清单20‑14\ **(3)**\ ：进入临界区，程序能执行到这里说明其他任务或中断向这个任务发送了一个任务通知，或者任务本身的阻塞超时时间到了，现在无论有没有任务通知都要继续处理。

代码清单20‑14\ **(4)**\ ：先获取一下任务通知值，因为现在并不知道任务通知是否有效，所以还是要再判断一下任务通知是否有效，有效则返回通知值，无效则退出，并且返回0，代表无效的任务通知值。

代码清单20‑14\ **(5)**\ ：如果任务通知有效，那在函数前判断一下是否要清除任务通知，根据用户指定的参数xClearCountOnExit处理，设置为pdFALSE时，函数xTaskNotifyTake()退出前，将任务的通知值减1，可以用来实现计数信号量；设置为pdTRUE时，函数xT
askNotifyTake()退出前，将任务通知值清零，可以用来实现二值信号量。

代码清单20‑14\ **(6)**\ ：不清除，那任务通知值就减1。

代码清单20‑14\ **(7)**\ ：恢复任务通知状态。

与获取二值信号量和获取计数信号量的函数相比，ulTaskNotifyTake()函数少了很多调用子函数开销、少了很多判断、少了事件列表处理、少了队列上锁与解锁处理等等，因此ulTaskNotifyTake()函数相对效率很高。

代码清单20‑15ulTaskNotifyTake()函数应用举例

1 /\* 中断服务程序：向一个任务发送任务通知 \*/

2 void vANInterruptHandler( void )

3 {

4 BaseType_t xHigherPriorityTaskWoken;

5

6 /\* 清除中断 \*/

7 prvClearInterruptSource();

8

9 /\* xHigherPriorityTaskWoken在使用之前必须设置为pdFALSE

10 如果调用vTaskNotifyGiveFromISR()会解除vHandlingTask任务的阻塞状态，

11 并且vHandlingTask任务的优先级高于当前处于运行状态的任务，

12 则xHigherPriorityTaskWoken将会自动被设置为pdTRUE \*/

13 xHigherPriorityTaskWoken = pdFALSE;

14

15 /\* 发送任务通知，并解锁阻塞在该任务通知下的任务 \*/

16 vTaskNotifyGiveFromISR( xHandlingTask, &xHigherPriorityTaskWoken );

17

18 /\* 如果被解锁的任务优先级比当前运行的任务的优先级高

19 则在中断退出前执行一次上下文切换，在中断退出后去执行

20 刚刚被唤醒的优先级更高的任务*/

21 portYIELD_FROM_ISR( xHigherPriorityTaskWoken );

22 }

23 /*-----------------------------------------------------------*/

24 /\* 任务：阻塞在一个任务通知上 \*/

25 void vHandlingTask( void \*pvParameters )

26 {

27 BaseType_t xEvent;

28

29 for ( ;; ) {

30 /\* 一直阻塞（没有时间限制，所以没有必要检测函数的返回值）

31 这里 RTOS 的任务通知值被用作二值信号量，所以在函数退出

32 时，任务通知值要被清0 。要注意的是真正的应用程序不应该

33 无限期的阻塞*/

34 ulTaskNotifyTake( pdTRUE, /\* 在退出前清0任务通知值 \*/

35 portMAX_DELAY ); /\* 无限阻塞 \*/

36

37 /\* RTOS 任务通知被当作二值信号量使用

38 当处理完所有的事情后继续等待下一个任务通知*/

39 do {

40 xEvent = xQueryPeripheral();

41

42 if ( xEvent != NO_MORE_EVENTS ) {

43 vProcessPeripheralEvent( xEvent );

44 }

45

46 } while ( xEvent != NO_MORE_EVENTS );

47 }

48 }

xTaskNotifyWait()
'''''''''''''''''

xTaskNotifyWait()函数用于实现全功能版的等待任务通知，根据用户指定的参数的不同，可以灵活的用于实现轻量级的消息队列队列、二值信号量、计数信号量和事件组功能，并带有超时等待。函数的具体说明见表20‑9，函数实现源码具体见代码清单20‑16。

表20‑9xTaskNotifyWait()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xTaskNotifyWait( uint32_t ulBitsToClearOnEntry,  uint32_t ulBitsToClearOnExit,  uint32_t \*pulNotificationValue,  TickType_t
       xTicksToWait );
     - |

   * - **功能**     |
     - 于等待一个任           | 务通知，并带有超时等待。 |
     - |

   * - **参数**     |
     - lBitsToClearOnEntry     |
     - lBitsToClearOnEntry表   | 示在使用通知之前，将任务 | 通知值的哪些位清0，实现  | 过程就是将任务的通知值与 | 参数ulBitsToClearOnEntry | 的按位取反值按位与操作。 |

        如果ulBitsToClearOnEntry | 设置为0x01，那么在函数进 | 入前，任务通知值的位1会  | 被清0，其他位保持不变。  | 如果ulBitsToClearOnEntry | 设置为 0xFFFFFFFF(ULONG_ | MAX)，那么在进入函数前任 |
        务通知值的所有位都会被清 | 0，表示清零任务通知值。  |

   * -
     - ulBitsToClearOnExit
     - u lBitsToClearOnExit表示在 | 函数xTaskNotifyWait()退  | 出前，决定任务接收到的通 | 知值的哪些位会被清0，实  | 现过程就是将任务的通知值 | 与参数ulBitsToClearOnEx  | it的按位取反值按位与操作 | 。在清0前，接收到的任务
       | 通知值会先被保存到形参*  | pulNotificationValue中。 |

       如果ulBitsToClearOnExit  | 设置为0x03，那么在函数退 | 出前，接收到的任务通知值 | 的位0和位1会被清0，其他  | 位保持不变。如果ulBitsTo | ClearOnExi设置为 0xFFFFF | FFF(ULONG_MAX)，那么在退 |
       出函数前接收到的任务通知 | 值的所有位都会被清0，表  | 示退出时清零任务通知值。 |

   * -
     - pulNotificationValue
     - 用于保存接收到的         | 任务通知值。如果接收到的 | 任务通知不需要使用，则设 | 置为NULL即可。这个通知值 | 在参数ulBitsToClearOnExi | t起作用前将通知值拷贝到* | pulNotificationValue中。 |

   * -
     - xTicksToWait
     - 等待超时时               | 间，单位为系统节拍周期。 | 宏pdMS_TO_TICKS用于将单  | 位毫秒转化为系统节拍数。 |

   * - **返回值**   | 如
     - 获                   | 取任务通知成功则返回pdT  | RUE，失败则返回pdFALSE。 |
     - |


代码清单20‑16xTaskNotifyWait()源码

1 #if( configUSE_TASK_NOTIFICATIONS == 1 )

2

3 BaseType_t xTaskNotifyWait( uint32_t ulBitsToClearOnEntry,

4 uint32_t ulBitsToClearOnExit,

5 uint32_t \*pulNotificationValue,

6 TickType_t xTicksToWait )

7 {

8 BaseType_t xReturn;

9

10 /\* 进入临界段 \*/

11 taskENTER_CRITICAL(); **(1)**

12 {

13 /\* 只有任务当前没有收到任务通知，才会将任务阻塞 \*/ **(2)**

14 if ( pxCurrentTCB->ucNotifyState != taskNOTIFICATION_RECEIVED ) {

15 /\* 使用任务通知值之前,根据用户指定参数ulBitsToClearOnEntryClear

16 将通知值的某些或全部位清零 \*/

17 pxCurrentTCB->ulNotifiedValue &= ~ulBitsToClearOnEntry;\ **(3)**

18

19 /\* 设置任务状态标识:等待通知 \*/

20 pxCurrentTCB->ucNotifyState = taskWAITING_NOTIFICATION;

21

22 /\* 挂起任务等待通知或者进入阻塞态 \*/

23 if ( xTicksToWait > ( TickType_t ) 0 ) { **(4)**

24 /\* 根据用户指定超时时间将任务添加到延时列表 \*/

25 prvAddCurrentTaskToDelayedList( xTicksToWait, pdTRUE );

26 traceTASK_NOTIFY_WAIT_BLOCK();

27

28 /\* 任务切换 \*/

29 portYIELD_WITHIN_API(); **(5)**

30 } else {

31 mtCOVERAGE_TEST_MARKER();

32 }

33 } else {

34 mtCOVERAGE_TEST_MARKER();

35 }

36 }

37 taskEXIT_CRITICAL();

38

39 //程序能执行到这里说明其他任务或中断向这个任务发送了通知或者任务阻塞超时,

40 现在继续处理

41

42 taskENTER_CRITICAL(); **(6)**

43 {

44 traceTASK_NOTIFY_WAIT();

45

46 if ( pulNotificationValue != NULL ) { **(7)**

47 /\* 返回当前通知值,通过指针参数传递 \*/

48 \*pulNotificationValue = pxCurrentTCB->ulNotifiedValue;

49 }

50

51 /\* 判断是否是因为任务阻塞超时，因为如果有

52 任务发送了通知的话，任务通知状态会被改变 \*/

53 if ( pxCurrentTCB->ucNotifyState == taskWAITING_NOTIFICATION ) {

54 /\* 没有收到任务通知,是阻塞超时 \*/

55 xReturn = pdFALSE; **(8)**

56 } else {

57 /\* 收到任务值,先将参数ulBitsToClearOnExit取反后与通知值位做按位与运算

58 在退出函数前,将通知值的某些或者全部位清零.
\*/

59 pxCurrentTCB->ulNotifiedValue &= ~ulBitsToClearOnExit;

60 xReturn = pdTRUE; **(9)**

61 }

62

63 //重新设置任务通知状态

64 pxCurrentTCB->ucNotifyState = taskNOT_WAITING_NOTIFICATION;\ **(10)**

65 }

66 taskEXIT_CRITICAL();

67

68 return xReturn;

69 }

70 #endif

代码清单20‑16\ **(1)**\ ：进入临界段。因为下面的操作可能会对任务的状态列表进行操作，系统不希望被打扰。

代码清单20‑16\ **(2)**\ ：只有任务当前没有收到任务通知，才会将任务阻塞，先看看任务通知是否有效，无效的话就将任务阻塞。

代码清单20‑16\ **(3)**\ ：使用任务通知值之前，根据用户指定参数ulBitsToClearOnEntryClear将通知值的某些或全部位清零。然后设置任务状态标识，表示当前任务在等待通知。

代码清单20‑16\ **(4)**\ ：如果用户指定了阻塞超时时间，那么系统将挂起任务等待通知或进入阻塞态，根据用户指定超时时间将任务添加到延时列表。

代码清单20‑16\ **(5)**\ ：然后进行任务切换。触发PendSV悬挂中断，在退出临界区的时候，进行任务切换。

代码清单20‑16\ **(6)**\ ：程序能执行到这里说明其他任务或中断向这个任务发送了通知或者任务阻塞超时，任务从阻塞态变成运行态，现在继续处理。

代码清单20‑16\ **(7)**\ ：返回当前通知值，通过指针参数传递。

代码清单20‑16\ **(8)**\ ：判断是否是因为任务阻塞超时才退出阻塞的，还是因为其他任务或中断发送了任务通知导致任务被恢复，为什么简单判断一下任务状态就知道？因为如果有任务发送了通知的话，任务通知状态会被改变，而阻塞退出的时候，任务通知状态还是原来的，现在看来是阻塞超时时间到来才恢复运行的
，并没有接收到如何通知，那么返回pdFALSE。

代码清单20‑16\ **(9)**\ ：收到任务值，先将参数 ulBitsToClearOnExit 取反后与通知值位做按位与运算，在退出函数前，将通知值的某些或者全部位清零。

代码清单20‑16\ **(10)**\ ：重新设置任务通知状态。

纵观整个任务通知的实现，我们不难发现它比消息队列、信号量、事件的实现方式要简单很多。它可以实现轻量级的消息队列、二值信号量、计数信号量和事件组，并且使用更方便、更节省RAM、更高效，xTaskNotifyWait()函数的使用很简单，具体见代码清单20‑17。

至此，任务通知的函数基本讲解完成，但是我们有必要说明一下，任务通知并不能完全代替队列、二值信号量、计数信号量和事件组，使用的时候需要用户按需处理，此外，再提一次任务通知的局限性：

-  只能有一个任务接收通知事件。

-  接收通知的任务可以因为等待通知而进入阻塞状态，但是发送通知的任务即便不能立即完成发送通知，也不能进入阻塞状态。

代码清单20‑17xTaskNotifyWait()函数使用实例

1 /\* 这个任务展示使用任务通知值的位来传递不同的事件

2 这在某些情况下可以代替事件标志组。*/

3 void vAnEventProcessingTask( void \*pvParameters )

4 {

5 uint32_t ulNotifiedValue;

6

7 for ( ;; ) {

8 /\* 等待任务通知，无限期阻塞（没有超时，所以没有必要检查函数返回值）

9 这个任务的任务通知值的位由标志事件发生的任务或者中断来设置*/

10 xTaskNotifyWait( 0x00, /\* 在进入的时候不清除通知值的任何位 \*/

11 ULONG_MAX, /\* 在退出的时候复位通知值为0 \*/

12 &ulNotifiedValue, /\* 任务通知值传递到变量

13 ulNotifiedValue中*/

14 portMAX_DELAY ); /\* 无限期等待 \*/

15

16

17 /\* 根据任务通知值里面的各个位的值处理事情 \*/

18 if ( ( ulNotifiedValue & 0x01 ) != 0 ) {

19 /\* 位0被置1 \*/

20 prvProcessBit0Event();

21 }

22

23 if ( ( ulNotifiedValue & 0x02 ) != 0 ) {

24 /\* 位1被置1 \*/

25 prvProcessBit1Event();

26 }

27

28 if ( ( ulNotifiedValue & 0x04 ) != 0 ) {

29 /\* 位2被置1 \*/

30 prvProcessBit2Event();

31 }

32

33 /\* ...
等等 \*/

34 }

35 }

任务通知实验
~~~~~~

任务通知代替消息队列
^^^^^^^^^^

任务通知代替消息队列是在FreeRTOS中创建了三个任务，其中两个任务是用于接收任务通知，另一个任务发送任务通知。三个任务独立运行，发送消息任务是通过检测按键的按下情况来发送消息通知，另两个任务获取消息通知，在任务通知中没有可用的通知之前就一直等待消息，一旦获取到消息通知就把消息打印在串口调试助手里
，具体见代码清单20‑18加粗部分。

代码清单20‑18任务通知代替消息队列

1 /\*

2 \\*

3 \* 包含的头文件

4 \\*

5 \*/

6 /\* FreeRTOS头文件 \*/

7 #include"FreeRTOS.h"

8 #include"task.h"

9 /\* 开发板硬件bsp头文件 \*/

10 #include"bsp_led.h"

11 #include"bsp_usart.h"

12 #include"bsp_key.h"

13 #include"limits.h"

14 /\* 任务句柄 \/

15 /\*

16 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

17 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

18 \* 这个句柄可以为NULL。

19 \*/

20 static TaskHandle_t AppTaskCreate_Handle = NULL;/*创建任务句柄 \*/

21 static TaskHandle_t Receive1_Task_Handle = NULL;/*Receive1_Task任务句柄 \*/

22 static TaskHandle_t Receive2_Task_Handle = NULL;/*Receive2_Task任务句柄 \*/

23 static TaskHandle_t Send_Task_Handle = NULL;/\* Send_Task任务句柄 \*/

24

25 /\* 内核对象句柄 \/

26 /\*

27 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

28 \* 对象，必须先创建，创建成功之后会返回一个相应的句柄。实际上就是一个指针，后续我

29 \* 们就可以通过这个句柄操作这些内核对象。

30 \*

31 \*

32 内核对象说白了就是一种全局的数据结构，通过这些数据结构我们可以实现任务间的通信，

33 \* 任务间的事件同步等各种功能。至于这些功能的实现我们是通过调用这些内核对象的函数

34 \* 来完成的

35 \*

36 \*/

37

38

39 /\* 全局变量声明 \/

40 /\*

41 \* 当我们在写应用程序的时候，可能需要用到一些全局变量。

42 \*/

43

44

45 /\* 宏定义 \/

46 /\*

47 \* 当我们在写应用程序的时候，可能需要用到一些宏定义。

48 \*/

49 #define USE_CHAR 0/\* 测试字符串的时候配置为 1 ，测试变量配置为 0 \*/

50

51 /\*

52 \\*

53 \* 函数声明

54 \\*

55 \*/

56 static void AppTaskCreate(void);/\* 用于创建任务 \*/

57

58 static void Receive1_Task(void\* pvParameters);/\* Receive1_Task任务实现 \*/

59 static void Receive2_Task(void\* pvParameters);/\* Receive2_Task任务实现 \*/

60

61 static void Send_Task(void\* pvParameters);/\* Send_Task任务实现 \*/

62

63 static void BSP_Init(void);/\* 用于初始化板载相关资源 \*/

64

65 /\*

66 \* @brief 主函数

67 \* @param 无

68 \* @retval 无

69 \* @note 第一步：开发板硬件初始化

70 第二步：创建APP应用任务

71 第三步：启动FreeRTOS，开始多任务调度

72 \/

73 int main(void)

74 {

75 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

76

77 /\* 开发板硬件初始化 \*/

78 BSP_Init();

79 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS任务通知代替消息队列实验！\n");

80 printf("按下KEY1或者KEY2向任务发送消息通知\n");

81 /\* 创建AppTaskCreate任务 \*/

82 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate, /*任务入口函数 \*/

83 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

84 (uint16_t )512, /\* 任务栈大小 \*/

85 (void\* )NULL,/\* 任务入口函数参数 \*/

86 (UBaseType_t )1, /\* 任务的优先级 \*/

87 (TaskHandle_t\* )&AppTaskCreate_Handle);/\* 任务控制块指针 \*/

88 /\* 启动任务调度 \*/

89 if (pdPASS == xReturn)

90 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

91 else

92 return -1;

93

94 while (1); /\* 正常不会执行到这里 \*/

95 }

96

97

98 /\*

99 \* @ 函数名： AppTaskCreate

100 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

101 \* @ 参数：无

102 \* @ 返回值：无

103 \/

104 static void AppTaskCreate(void)

105 {

106 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

107

108 taskENTER_CRITICAL(); //进入临界区

109

110 /\* 创建Receive1_Task任务 \*/

111 xReturn = xTaskCreate((TaskFunction_t )Receive1_Task,/*任务入口函数 \*/

112 (const char\* )"Receive1_Task",/\* 任务名字 \*/

113 (uint16_t )512, /\* 任务栈大小 \*/

114 (void\* )NULL, /\* 任务入口函数参数 \*/

115 (UBaseType_t )2, /\* 任务的优先级 \*/

116 (TaskHandle_t*)&Receive1_Task_Handle);/*任务控制块指针 \*/

117 if (pdPASS == xReturn)

118 printf("创建Receive1_Task任务成功!\r\n");

119

120 /\* 创建Receive2_Task任务 \*/

121 xReturn = xTaskCreate((TaskFunction_t )Receive2_Task, /\* 任务入口函数 \*/

122 (const char\* )"Receive2_Task",/\* 任务名字 \*/

123 (uint16_t )512, /\* 任务栈大小 \*/

124 (void\* )NULL, /\* 任务入口函数参数 \*/

125 (UBaseType_t )3, /\* 任务的优先级 \*/

126 (TaskHandle_t*)&Receive2_Task_Handle);/*任务控制块指针 \*/

127 if (pdPASS == xReturn)

128 printf("创建Receive2_Task任务成功!\r\n");

129

130 /\* 创建Send_Task任务 \*/

131 xReturn = xTaskCreate((TaskFunction_t )Send_Task, /\* 任务入口函数 \*/

132 (const char\* )"Send_Task",/\* 任务名字 \*/

133 (uint16_t )512, /\* 任务栈大小 \*/

134 (void\* )NULL,/\* 任务入口函数参数 \*/

135 (UBaseType_t )4, /\* 任务的优先级 \*/

136 (TaskHandle_t\* )&Send_Task_Handle);/\* 任务控制块指针 \*/

137 if (pdPASS == xReturn)

138 printf("创建Send_Task任务成功!\n\n");

139

140 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

141

142 taskEXIT_CRITICAL(); //退出临界区

143 }

144

145

146

147 /\*

148 \* @ 函数名： Receive_Task

149 \* @ 功能说明： Receive_Task任务主体

150 \* @ 参数：

151 \* @ 返回值：无

152 \/

**153 static void Receive1_Task(void\* parameter)**

**154 {**

**155 BaseType_t xReturn = pdTRUE;/\* 定义一个创建信息返回值，默认为pdPASS \*/**

**156 #if USE_CHAR**

**157 char \*r_char;**

**158 #else**

**159 uint32_t r_num;**

**160 #endif**

**161 while (1) {**

**162 //获取任务通知 ,没获取到则一直等待**

**163 xReturn=xTaskNotifyWait(0x0, //进入函数的时候不清除任务bit**

**164 ULONG_MAX,//退出函数的时候清除所有的bit**

**165 #if USE_CHAR**

**166 (uint32_t \*)&r_char,//保存任务通知值**

**167 #else**

**168 &r_num, //保存任务通知值**

**169 #endif**

**170 portMAX_DELAY); //阻塞时间**

**171 if ( pdTRUE == xReturn )**

**172 #if USE_CHAR**

**173 printf("Receive1_Task 任务通知为 %s\n",r_char);**

**174 #else**

**175 printf("Receive1_Task 任务通知为 %d\n",r_num);**

**176 #endif**

**177**

**178**

**179 LED1_TOGGLE;**

**180 }**

**181 }**

182

183 /\*

184 \* @ 函数名： Receive_Task

185 \* @ 功能说明： Receive_Task任务主体

186 \* @ 参数：

187 \* @ 返回值：无

188 \/

**189 static void Receive2_Task(void\* parameter)**

**190 {**

**191 BaseType_t xReturn = pdTRUE;/\* 定义一个创建信息返回值，默认为pdPASS \*/**

**192 #if USE_CHAR**

**193 char \*r_char;**

**194 #else**

**195 uint32_t r_num;**

**196 #endif**

**197 while (1) {**

**198 //获取任务通知 ,没获取到则一直等待**

**199 xReturn=xTaskNotifyWait(0x0, //进入函数的时候不清除任务bit**

**200 ULONG_MAX, //退出函数的时候清除所有的bit**

**201 #if USE_CHAR**

**202 (uint32_t \*)&r_char, //保存任务通知值**

**203 #else**

**204 &r_num, //保存任务通知值**

**205 #endif**

**206 portMAX_DELAY); //阻塞时间**

**207 if ( pdTRUE == xReturn )**

**208 #if USE_CHAR**

**209 printf("Receive2_Task 任务通知为 %s\n",r_char);**

**210 #else**

**211 printf("Receive2_Task 任务通知为 %d\n",r_num);**

**212 #endif**

**213 LED2_TOGGLE;**

**214 }**

**215 }**

216

217 /\*

218 \* @ 函数名： Send_Task

219 \* @ 功能说明： Send_Task任务主体

220 \* @ 参数：

221 \* @ 返回值：无

222 \/

**223 static void Send_Task(void\* parameter)**

**224 {**

**225 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/**

**226 #if USE_CHAR**

**227 char test_str1[] = "this is a mail test 1";/*消息test1 \*/**

**228 char test_str2[] = "this is a mail test 2";/*消息test2 \*/**

**229 #else**

**230 uint32_t send1 = 1;**

**231 uint32_t send2 = 2;**

**232 #endif**

**233**

**234**

**235**

**236 while (1) {**

**237 /\* KEY1 被按下 \*/**

**238 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**239**

**240 xReturn = xTaskNotify( Receive1_Task_Handle, /*任务句柄*/**

**241 #if USE_CHAR**

**242 (uint32_t)&test_str1, /*发送的数据，最大为4字节 \*/**

**243 #else**

**244 send1, /\* 发送的数据，最大为4字节 \*/**

**245 #endif**

**246 eSetValueWithOverwrite );/*覆盖当前通知*/**

**247**

**248 if ( xReturn == pdPASS )**

**249 printf("Receive1_Task_Handle 任务通知释放成功!\r\n");**

**250 }**

**251 /\* KEY2 被按下 \*/**

**252 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**253 xReturn = xTaskNotify( Receive2_Task_Handle, /*任务句柄*/**

**254 #if USE_CHAR**

**255 (uint32_t)&test_str2,/\* 发送的数据，最大为4字节 \*/**

**256 #else**

**257 send2, /\* 发送的数据，最大为4字节 \*/**

**258 #endif**

**259 eSetValueWithOverwrite );/*覆盖当前通知*/**

**260 /\* 此函数只会返回pdPASS \*/**

**261 if ( xReturn == pdPASS )**

**262 printf("Receive2_Task_Handle 任务通知释放成功!\r\n");**

**263 }**

**264 vTaskDelay(20);**

**265 }**

**266 }**

267 /\*

268 \* @ 函数名： BSP_Init

269 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

270 \* @ 参数：

271 \* @ 返回值：无

272 \/

273 static void BSP_Init(void)

274 {

275 /\*

276 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

277 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

278 \* 都统一用这个优先级分组，千万不要再分组，切忌。

279 \*/

280 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

281

282 /\* LED 初始化 \*/

283 LED_GPIO_Config();

284

285 /\* 串口初始化 \*/

286 USART_Config();

287

288 /\* 按键初始化 \*/

289 Key_GPIO_Config();

290

291 }

292

293/END OF FILE/

任务通知代替二值信号量
^^^^^^^^^^^

任务通知代替消息队列是在FreeRTOS中创建了三个任务，其中两个任务是用于接收任务通知，另一个任务发送任务通知。三个任务独立运行，发送通知任务是通过检测按键的按下情况来发送通知，另两个任务获取通知，在任务通知中没有可用的通知之前就一直等待任务通知，获取到通知以后就将通知值清0，这样子是为了代替二值
信号量，任务同步成功则继续执行，然后在串口调试助手里将运行信息打印出来，具体见代码清单20‑19加粗部分。

代码清单20‑19任务通知代替二值信号量

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS V9.0.0 + STM32 二值信号量同步

8 \\*

9 \* @attention

10 \*

11 \* 实验平台:野火 STM32 开发板

12 \* 论坛 :http://www.firebbs.cn

13 \* 淘宝 :https://fire-stm32.taobao.com

14 \*

15 \\*

16 \*/

17

18 /\*

19 \\*

20 \* 包含的头文件

21 \\*

22 \*/

23 /\* FreeRTOS头文件 \*/

24 #include"FreeRTOS.h"

25 #include"task.h"

26 /\* 开发板硬件bsp头文件 \*/

27 #include"bsp_led.h"

28 #include"bsp_usart.h"

29 #include"bsp_key.h"

30 /\* 任务句柄 \/

31 /\*

32 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

33 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

34 \* 这个句柄可以为NULL。

35 \*/

36 static TaskHandle_t AppTaskCreate_Handle = NULL;/\* 创建任务句柄 \*/

37 static TaskHandle_t Receive1_Task_Handle = NULL;/*Receive1_Task任务句柄 \*/

38 static TaskHandle_t Receive2_Task_Handle = NULL;/*Receive2_Task任务句柄 \*/

39 static TaskHandle_t Send_Task_Handle = NULL;/\* Send_Task任务句柄 \*/

40

41 /\* 内核对象句柄 \/

42 /\*

43 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

44 \* 对象，必须先创建，创建成功之后会返回一个相应的句柄。实际上就是一个指针，后续我

45 \* 们就可以通过这个句柄操作这些内核对象。

46 \*

47 \*

48 内核对象说白了就是一种全局的数据结构，通过这些数据结构我们可以实现任务间的通信，

49 \* 任务间的事件同步等各种功能。至于这些功能的实现我们是通过调用这些内核对象的函数

50 \* 来完成的

51 \*

52 \*/

53

54

55 /\* 全局变量声明 \/

56 /\*

57 \* 当我们在写应用程序的时候，可能需要用到一些全局变量。

58 \*/

59

60

61 /\* 宏定义 \/

62 /\*

63 \* 当我们在写应用程序的时候，可能需要用到一些宏定义。

64 \*/

65

66

67 /\*

68 \\*

69 \* 函数声明

70 \\*

71 \*/

72 static void AppTaskCreate(void);/\* 用于创建任务 \*/

73

74 static void Receive1_Task(void\* pvParameters);/\* Receive1_Task任务实现 \*/

75 static void Receive2_Task(void\* pvParameters);/\* Receive2_Task任务实现 \*/

76

77 static void Send_Task(void\* pvParameters);/\* Send_Task任务实现 \*/

78

79 static void BSP_Init(void);/\* 用于初始化板载相关资源 \*/

80

81 /\*

82 \* @brief 主函数

83 \* @param 无

84 \* @retval 无

85 \* @note 第一步：开发板硬件初始化

86 第二步：创建APP应用任务

87 第三步：启动FreeRTOS，开始多任务调度

88 \/

89 int main(void)

90 {

91 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

92

93 /\* 开发板硬件初始化 \*/

94 BSP_Init();

95 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS任务通知代替二值信号量实验！\n");

96 printf("按下KEY1或者KEY2进行任务与任务间的同步\n");

97 /\* 创建AppTaskCreate任务 \*/

98 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,/\* 任务入口函数 \*/

99 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

100 (uint16_t )512, /\* 任务栈大小 \*/

101 (void\* )NULL,/\* 任务入口函数参数 \*/

102 (UBaseType_t )1, /\* 任务的优先级 \*/

103 (TaskHandle_t*)&AppTaskCreate_Handle);/\* 任务控制块指针 \*/

104 /\* 启动任务调度 \*/

105 if (pdPASS == xReturn)

106 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

107 else

108 return -1;

109

110 while (1); /\* 正常不会执行到这里 \*/

111 }

112

113

114 /\*

115 \* @ 函数名： AppTaskCreate

116 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

117 \* @ 参数：无

118 \* @ 返回值：无

119 \/

120 static void AppTaskCreate(void)

121 {

122 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

123

124 taskENTER_CRITICAL(); //进入临界区

125

126 /\* 创建Receive1_Task任务 \*/

127 xReturn = xTaskCreate((TaskFunction_t )Receive1_Task,/*任务入口函数 \*/

128 (const char\* )"Receive1_Task",/\* 任务名字 \*/

129 (uint16_t )512, /\* 任务栈大小 \*/

130 (void\* )NULL, /\* 任务入口函数参数 \*/

131 (UBaseType_t )2, /\* 任务的优先级 \*/

132 (TaskHandle_t\* )&Receive1_Task_Handle);/*任务控制块指针 \*/

133 if (pdPASS == xReturn)

134 printf("创建Receive1_Task任务成功!\r\n");

135

136 /\* 创建Receive2_Task任务 \*/

137 xReturn = xTaskCreate((TaskFunction_t )Receive2_Task,/*任务入口函数 \*/

138 (const char\* )"Receive2_Task",/\* 任务名字 \*/

139 (uint16_t )512, /\* 任务栈大小 \*/

140 (void\* )NULL, /\* 任务入口函数参数 \*/

141 (UBaseType_t )3, /\* 任务的优先级 \*/

142 (TaskHandle_t*)&Receive2_Task_Handle);/\* 任务控制块指针 \*/

143 if (pdPASS == xReturn)

144 printf("创建Receive2_Task任务成功!\r\n");

145

146 /\* 创建Send_Task任务 \*/

147 xReturn = xTaskCreate((TaskFunction_t )Send_Task, /\* 任务入口函数 \*/

148 (const char\* )"Send_Task",/\* 任务名字 \*/

149 (uint16_t )512, /\* 任务栈大小 \*/

150 (void\* )NULL,/\* 任务入口函数参数 \*/

151 (UBaseType_t )4, /\* 任务的优先级 \*/

152 (TaskHandle_t\* )&Send_Task_Handle);/\* 任务控制块指针 \*/

153 if (pdPASS == xReturn)

154 printf("创建Send_Task任务成功!\n\n");

155

156 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

157

158 taskEXIT_CRITICAL(); //退出临界区

159 }

160

161

162

163 /\*

164 \* @ 函数名： Receive_Task

165 \* @ 功能说明： Receive_Task任务主体

166 \* @ 参数：

167 \* @ 返回值：无

168 \/

**169 static void Receive1_Task(void\* parameter)**

**170 {**

**171 while (1) {**

**172 /\* uint32_t ulTaskNotifyTake(BaseType_t xClearCountOnExit,TickType_tTicksToWait );**

**173 \* xClearCountOnExit：pdTRUE 在退出函数的时候任务任务通知值清零，类似二值信号量**

**174 \* pdFALSE 在退出函数ulTaskNotifyTakeO的时候任务通知值减一，类似计数型信号量。**

**175 \*/**

**176 //获取任务通知 ,没获取到则一直等待**

**177 ulTaskNotifyTake(pdTRUE,portMAX_DELAY);**

**178**

**179 printf("Receive1_Task 任务通知获取成功!\n\n");**

**180**

**181 LED1_TOGGLE;**

**182 }**

**183 }**

184

185 /\*

186 \* @ 函数名： Receive_Task

187 \* @ 功能说明： Receive_Task任务主体

188 \* @ 参数：

189 \* @ 返回值：无

190 \/

**191 static void Receive2_Task(void\* parameter)**

**192 {**

**193 while (1) {**

**194 /*uint32_t ulTaskNotifyTake(BaseType_t xClearCountOnExit,TickType_txTicksToWait );**

**195 \* xClearCountOnExit：pdTRUE 在退出函数的时候任务任务通知值清零，类似二值信号量**

**196 \* pdFALSE 在退出函数ulTaskNotifyTakeO的时候任务通知值减一，类似计数型信号量。**

**197 \*/**

**198 //获取任务通知 ,没获取到则一直等待**

**199 ulTaskNotifyTake(pdTRUE,portMAX_DELAY);**

**200**

**201 printf("Receive2_Task 任务通知获取成功!\n\n");**

**202**

**203 LED2_TOGGLE;**

**204 }**

**205 }**

206

207 /\*

208 \* @ 函数名： Send_Task

209 \* @ 功能说明： Send_Task任务主体

210 \* @ 参数：

211 \* @ 返回值：无

212 \/

**213 static void Send_Task(void\* parameter)**

**214 {**

**215 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/**

**216 while (1) {**

**217 /\* KEY1 被按下 \*/**

**218 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**219 /\* 原型:BaseType_t xTaskNotifyGive( TaskHandle_t xTaskToNotify ); \*/**

**220 xReturn = xTaskNotifyGive(Receive1_Task_Handle);**

**221 /\* 此函数只会返回pdPASS \*/**

**222 if ( xReturn == pdTRUE )**

**223 printf("Receive1_Task_Handle 任务通知释放成功!\r\n");**

**224 }**

**225 /\* KEY2 被按下 \*/**

**226 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**227 xReturn = xTaskNotifyGive(Receive2_Task_Handle);**

**228 /\* 此函数只会返回pdPASS \*/**

**229 if ( xReturn == pdPASS )**

**230 printf("Receive2_Task_Handle 任务通知释放成功!\r\n");**

**231 }**

**232 vTaskDelay(20);**

**233 }**

**234 }**

235 /\*

236 \* @ 函数名： BSP_Init

237 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

238 \* @ 参数：

239 \* @ 返回值：无

240 \/

241 static void BSP_Init(void)

242 {

243 /\*

244 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

245 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

246 \* 都统一用这个优先级分组，千万不要再分组，切忌。

247 \*/

248 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

249

250 /\* LED 初始化 \*/

251 LED_GPIO_Config();

252

253 /\* 串口初始化 \*/

254 USART_Config();

255

256 /\* 按键初始化 \*/

257 Key_GPIO_Config();

258

259 }

260

261 /END OF FILE/

任务通知代替计数信号量
^^^^^^^^^^^

任务通知代替计数信号量是基于计数型信号量实验修改而来，模拟停车场工作运行。并且在FreeRTOS中创建了两个任务：一个是获取任务通知，一个是发送任务通知，两个任务独立运行，获取通知的任务是通过按下KEY1按键获取，模拟停车场停车操作，其等待时间是0；发送通知的任务则是通过检测KEY2按键按下进行通知
的发送，模拟停车场取车操作，并且在串口调试助手输出相应信息，实验源码具体见加粗部分。

代码清单20‑20任务通知代替计数信号量

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS V9.0.0 + STM32 任务通知实验

8 \\*

9 \* @attention

10 \*

11 \* 实验平台:野火 STM32 开发板

12 \* 论坛 :http://www.firebbs.cn

13 \* 淘宝 :https://fire-stm32.taobao.com

14 \*

15 \\*

16 \*/

17

18 /\*

19 \\*

20 \* 包含的头文件

21 \\*

22 \*/

23 /\* FreeRTOS头文件 \*/

24 #include"FreeRTOS.h"

25 #include"task.h"

26 #include"queue.h"

27 #include"semphr.h"

28 /\* 开发板硬件bsp头文件 \*/

29 #include"bsp_led.h"

30 #include"bsp_usart.h"

31 #include"bsp_key.h"

32 /\* 任务句柄 \/

33 /\*

34 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

35 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

36 \* 这个句柄可以为NULL。

37 \*/

38 static TaskHandle_t AppTaskCreate_Handle = NULL;/\* 创建任务句柄 \*/

39 static TaskHandle_t Take_Task_Handle = NULL;/\* Take_Task任务句柄 \*/

40 static TaskHandle_t Give_Task_Handle = NULL;/\* Give_Task任务句柄 \*/

41

42 /\* 内核对象句柄 \/

43 /\*

44 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

45 \* 对象，必须先创建，创建成功之后会返回一个相应的句柄。实际上就是一个指针，后续我

46 \* 们就可以通过这个句柄操作这些内核对象。

47 \*

48 \*

49 内核对象说白了就是一种全局的数据结构，通过这些数据结构我们可以实现任务间的通信，

50 \* 任务间的事件同步等各种功能。至于这些功能的实现我们是通过调用这些内核对象的函数

51 \* 来完成的

52 \*

53 \*/

54 SemaphoreHandle_t CountSem_Handle =NULL;

55

56 /\* 全局变量声明 \/

57 /\*

58 \* 当我们在写应用程序的时候，可能需要用到一些全局变量。

59 \*/

60

61

62 /\* 宏定义 \/

63 /\*

64 \* 当我们在写应用程序的时候，可能需要用到一些宏定义。

65 \*/

66

67

68 /\*

69 \\*

70 \* 函数声明

71 \\*

72 \*/

73 static void AppTaskCreate(void);/\* 用于创建任务 \*/

74

75 static void Take_Task(void\* pvParameters);/\* Take_Task任务实现 \*/

76 static void Give_Task(void\* pvParameters);/\* Give_Task任务实现 \*/

77

78 static void BSP_Init(void);/\* 用于初始化板载相关资源 \*/

79

80 /\*

81 \* @brief 主函数

82 \* @param 无

83 \* @retval 无

84 \* @note 第一步：开发板硬件初始化

85 第二步：创建APP应用任务

86 第三步：启动FreeRTOS，开始多任务调度

87 \/

88 int main(void)

89 {

90 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

91

92 /\* 开发板硬件初始化 \*/

93 BSP_Init();

94

95 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS任务通知-计数信号量实验！\n");

96 printf("车位默认值为0个，按下KEY1申请车位，按下KEY2释放车位！\n\n");

97

98 /\* 创建AppTaskCreate任务 \*/

99 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,/\* 任务入口函数 \*/

100 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

101 (uint16_t )512, /\* 任务栈大小 \*/

102 (void\* )NULL,/\* 任务入口函数参数 \*/

103 (UBaseType_t )1, /\* 任务的优先级 \*/

104 (TaskHandle_t\* )&AppTaskCreate_Handle);/*任务控制块指针 \*/

105 /\* 启动任务调度 \*/

106 if (pdPASS == xReturn)

107 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

108 else

109 return -1;

110

111 while (1); /\* 正常不会执行到这里 \*/

112 }

113

114

115 /\*

116 \* @ 函数名： AppTaskCreate

117 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

118 \* @ 参数：无

119 \* @ 返回值：无

120 \/

121 static void AppTaskCreate(void)

122 {

123 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

124

125 taskENTER_CRITICAL(); //进入临界区

126

127 /\* 创建Take_Task任务 \*/

128 xReturn = xTaskCreate((TaskFunction_t )Take_Task, /\* 任务入口函数 \*/

129 (const char\* )"Take_Task",/\* 任务名字 \*/

130 (uint16_t )512, /\* 任务栈大小 \*/

131 (void\* )NULL, /\* 任务入口函数参数 \*/

132 (UBaseType_t )2, /\* 任务的优先级 \*/

133 (TaskHandle_t\* )&Take_Task_Handle);/\* 任务控制块指针 \*/

134 if (pdPASS == xReturn)

135 printf("创建Take_Task任务成功!\r\n");

136

137 /\* 创建Give_Task任务 \*/

138 xReturn = xTaskCreate((TaskFunction_t )Give_Task, /\* 任务入口函数 \*/

139 (const char\* )"Give_Task",/\* 任务名字 \*/

140 (uint16_t )512, /\* 任务栈大小 \*/

141 (void\* )NULL,/\* 任务入口函数参数 \*/

142 (UBaseType_t )3, /\* 任务的优先级 \*/

143 (TaskHandle_t\* )&Give_Task_Handle);/\* 任务控制块指针 \*/

144 if (pdPASS == xReturn)

145 printf("创建Give_Task任务成功!\n\n");

146

147 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

148

149 taskEXIT_CRITICAL(); //退出临界区

150 }

151

152

153

154 /\*

155 \* @ 函数名： Take_Task

156 \* @ 功能说明： Take_Task任务主体

157 \* @ 参数：

158 \* @ 返回值：无

159 \/

**160 static void Take_Task(void\* parameter)**

**161 {**

**162 uint32_t take_num = pdTRUE;/\* 定义一个创建信息返回值，默认为pdPASS \*/**

**163 /\* 任务都是一个无限循环，不能返回 \*/**

**164 while (1) {**

**165 //如果KEY1被按下**

**166 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**167 /*uint32_t ulTaskNotifyTake(BaseType_t xClearCountOnExit,TickType_t xTicksToWait );**

**168 \* xClearCountOnExit：pdTRUE 在退出函数的时候任务任务通知值清零，类似二值信号量**

**169 \* pdFALSE 在退出函数ulTaskNotifyTakeO的时候任务通知值减一，类似计数型信号量。**

**170 \*/**

**171 //获取任务通知 ,没获取到则不等待**

**172 take_num=ulTaskNotifyTake(pdFALSE,0);//**

**173 if (take_num > 0)**

**174 printf( "KEY1被按下，成功申请到停车位。当前车位为 %d\n", take_num
- 1);**

**175 else**

**176 printf( "KEY1被按下，车位已经没有了。按KEY2释放车位\n" );**

**177 }**

**178 vTaskDelay(20); //每20ms扫描一次**

**179 }**

**180 }**

181

182 /\*

183 \* @ 函数名： Give_Task

184 \* @ 功能说明： Give_Task任务主体

185 \* @ 参数：

186 \* @ 返回值：无

187 \/

**188 static void Give_Task(void\* parameter)**

**189 {**

**190 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/**

**191 /\* 任务都是一个无限循环，不能返回 \*/**

**192 while (1) {**

**193 //如果KEY2被按下**

**194 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**195**

**196 /\* 释放一个任务通知 \*/**

**197 xTaskNotifyGive(Take_Task_Handle);//发送任务通知**

**198 /\* 此函数只会返回pdPASS \*/**

**199 if ( pdPASS == xReturn )**

**200 printf( "KEY2被按下，释放1个停车位。\n" );**

**201 }**

**202 vTaskDelay(20); //每20ms扫描一次**

**203 }**

**204 }**

205 /\*

206 \* @ 函数名： BSP_Init

207 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

208 \* @ 参数：

209 \* @ 返回值：无

210 \/

211 static void BSP_Init(void)

212 {

213 /\*

214 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

215 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

216 \* 都统一用这个优先级分组，千万不要再分组，切忌。

217 \*/

218 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

219

220 /\* LED 初始化 \*/

221 LED_GPIO_Config();

222

223 /\* 串口初始化 \*/

224 USART_Config();

225

226 /\* 按键初始化 \*/

227 Key_GPIO_Config();

228

229 }

230

231 /END OF FILE/

任务通知代替事件组
^^^^^^^^^

任务通知代替事件组实验是在事件标志组实验基础上进行修改，实验任务通知替代事件实现事件类型的通信，该实验是在FreeRTOS中创建了两个任务，一个是发送事件通知任务，一个是等待事件通知任务，两个任务独立运行，发送事件通知任务通过检测按键的按下情况设置不同的通知值位，等待事件通知任务则获取这任务通知值，
并且根据通知值判断两个事件是否都发生，如果是则输出相应信息，LED进行翻转。等待事件通知任务的等待时间是portMAX_DELAY，一直在等待事件通知的发生，等待获取到事件之后清除对应的任务通知值的位，具体见代码清单20‑21加粗部分。

代码清单20‑21任务通知代替事件组

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS V9.0.0 + STM32 任务通知替代事件

8 \\*

9 \* @attention

10 \*

11 \* 实验平台:野火 STM32 开发板

12 \* 论坛 :http://www.firebbs.cn

13 \* 淘宝 :https://fire-stm32.taobao.com

14 \*

15 \\*

16 \*/

17

18 /\*

19 \\*

20 \* 包含的头文件

21 \\*

22 \*/

23 /\* FreeRTOS头文件 \*/

24 #include"FreeRTOS.h"

25 #include"task.h"

26 #include"event_groups.h"

27 /\* 开发板硬件bsp头文件 \*/

28 #include"bsp_led.h"

29 #include"bsp_usart.h"

30 #include"bsp_key.h"

31 #include"limits.h"

32 /\* 任务句柄 \/

33 /\*

34 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

35 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

36 \* 这个句柄可以为NULL。

37 \*/

38 static TaskHandle_t AppTaskCreate_Handle = NULL;/\* 创建任务句柄 \*/

39 static TaskHandle_t LED_Task_Handle = NULL;/\* LED_Task任务句柄 \*/

40 static TaskHandle_t KEY_Task_Handle = NULL;/\* KEY_Task任务句柄 \*/

41

42 /\* 内核对象句柄 \/

43 /\*

44 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

45 \* 对象，必须先创建，创建成功之后会返回一个相应的句柄。实际上就是一个指针，后续我

46 \* 们就可以通过这个句柄操作这些内核对象。

47 \*

48 \*

49 内核对象说白了就是一种全局的数据结构，通过这些数据结构我们可以实现任务间的通信，

50 \* 任务间的事件同步等各种功能。至于这些功能的实现我们是通过调用这些内核对象的函数

51 \* 来完成的

52 \*

53 \*/

54 static EventGroupHandle_t Event_Handle =NULL;

55

56 /\* 全局变量声明 \/

57 /\*

58 \* 当我们在写应用程序的时候，可能需要用到一些全局变量。

59 \*/

60

61

62 /\* 宏定义 \/

63 /\*

64 \* 当我们在写应用程序的时候，可能需要用到一些宏定义。

65 \*/

**66 #define KEY1_EVENT (0x01 << 0)//设置事件掩码的位0**

**67 #define KEY2_EVENT (0x01 << 1)//设置事件掩码的位1**

68

69 /\*

70 \\*

71 \* 函数声明

72 \\*

73 \*/

74 static void AppTaskCreate(void);/\* 用于创建任务 \*/

75

76 static void LED_Task(void\* pvParameters);/\* LED_Task 任务实现 \*/

77 static void KEY_Task(void\* pvParameters);/\* KEY_Task 任务实现 \*/

78

79 static void BSP_Init(void);/\* 用于初始化板载相关资源 \*/

80

81 /\*

82 \* @brief 主函数

83 \* @param 无

84 \* @retval 无

85 \* @note 第一步：开发板硬件初始化

86 第二步：创建APP应用任务

87 第三步：启动FreeRTOS，开始多任务调度

88 \/

89 int main(void)

90 {

91 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

92

93 /\* 开发板硬件初始化 \*/

94 BSP_Init();

95 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS优先级翻转实验！\n");

96 printf("按下KEY1|KEY2发送任务事件通知！\n");

97 /\* 创建AppTaskCreate任务 \*/

98 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,/\* 任务入口函数 \*/

99 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

100 (uint16_t )512, /\* 任务栈大小 \*/

101 (void\* )NULL,/\* 任务入口函数参数 \*/

102 (UBaseType_t )1, /\* 任务的优先级 \*/

103 (TaskHandle_t*)&AppTaskCreate_Handle);/\* 任务控制块指针 \*/

104 /\* 启动任务调度 \*/

105 if (pdPASS == xReturn)

106 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

107 else

108 return -1;

109

110 while (1); /\* 正常不会执行到这里 \*/

111 }

112

113

114 /\*

115 \* @ 函数名： AppTaskCreate

116 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

117 \* @ 参数：无

118 \* @ 返回值：无

119 \/

120 static void AppTaskCreate(void)

121 {

122 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

123

124 taskENTER_CRITICAL(); //进入临界区

125

126 /\* 创建 Event_Handle \*/

127 Event_Handle = xEventGroupCreate();

128 if (NULL != Event_Handle)

129 printf("Event_Handle 事件创建成功!\r\n");

130

131 /\* 创建LED_Task任务 \*/

132 xReturn = xTaskCreate((TaskFunction_t )LED_Task, /\* 任务入口函数 \*/

133 (const char\* )"LED_Task",/\* 任务名字 \*/

134 (uint16_t )512, /\* 任务栈大小 \*/

135 (void\* )NULL, /\* 任务入口函数参数 \*/

136 (UBaseType_t )2, /\* 任务的优先级 \*/

137 (TaskHandle_t\* )&LED_Task_Handle);/\* 任务控制块指针 \*/

138 if (pdPASS == xReturn)

139 printf("创建LED_Task任务成功!\r\n");

140

141 /\* 创建KEY_Task任务 \*/

142 xReturn = xTaskCreate((TaskFunction_t )KEY_Task, /\* 任务入口函数 \*/

143 (const char\* )"KEY_Task",/\* 任务名字 \*/

144 (uint16_t )512, /\* 任务栈大小 \*/

145 (void\* )NULL,/\* 任务入口函数参数 \*/

146 (UBaseType_t )3, /\* 任务的优先级 \*/

147 (TaskHandle_t\* )&KEY_Task_Handle);/\* 任务控制块指针 \*/

148 if (pdPASS == xReturn)

149 printf("创建KEY_Task任务成功!\n");

150

151 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

152

153 taskEXIT_CRITICAL(); //退出临界区

154 }

155

156

157

158 /\*

159 \* @ 函数名： LED_Task

160 \* @ 功能说明： LED_Task任务主体

161 \* @ 参数：

162 \* @ 返回值：无

163 \/

**164 static void LED_Task(void\* parameter)**

**165 {**

**166 uint32_t r_event = 0; /\* 定义一个事件接收变量 \*/**

**167 uint32_t last_event = 0;/\* 定义一个保存事件的变量 \*/**

**168 BaseType_t xReturn = pdTRUE;/\* 定义一个创建信息返回值，默认为pdPASS \*/**

**169 /\* 任务都是一个无限循环，不能返回 \*/**

**170 while (1) {**

**171 /\* BaseType_t xTaskNotifyWait(uint32_t ulBitsToClearOnEntry,**

**172 uint32_t ulBitsToClearOnExit,**

**173 uint32_t \*pulNotificationValue,**

**174 TickType_t xTicksToWait );**

**175 \* ulBitsToClearOnEntry：当没有接收到任务通知的时候将任务通知值与此参数的取**

**176 反值进行按位与运算，当此参数为Oxfffff或者ULONG_MAX的时候就会将任务通知值清零。**

**177 \* ulBits ToClearOnExit：如果接收到了任务通知，在做完相应的处理退出函数之前将**

**178 任务通知值与此参数的取反值进行按位与运算，当此参数为0xfffff或者ULONG MAX的时候**

**179 就会将任务通知值清零。**

**180 \* pulNotification Value：此参数用来保存任务通知值。**

**181 \* xTick ToWait：阻塞时间。**

**182 \**

**183 \* 返回值：pdTRUE：获取到了任务通知。pdFALSE：任务通知获取失败。**

**184 \*/**

**185 //获取任务通知 ,没获取到则一直等待**

**186 xReturn = xTaskNotifyWait(0x0, //进入函数的时候不清除任务bit**

**187 ULONG_MAX, //退出函数的时候清除所有的bitR**

**188 &r_event, //保存任务通知值**

**189 portMAX_DELAY);//阻塞时间**

**190 if ( pdTRUE == xReturn ) {**

**191 last_event \|= r_event;**

**192 /\* 如果接收完成并且正确 \*/**

**193 if (last_event == (KEY1_EVENT|KEY2_EVENT)) {**

**194 last_event = 0; /\* 上一次的事件清零 \*/**

**195 printf ( "Key1与Key2都按下\n");**

**196 LED1_TOGGLE; //LED1 反转**

**197 } else/\* 否则就更新事件 \*/**

**198 last_event = r_event; /\* 更新上一次触发的事件 \*/**

**199 }**

**200**

**201 }**

**202 }**

203

204 /\*

205 \* @ 函数名： KEY_Task

206 \* @ 功能说明： KEY_Task任务主体

207 \* @ 参数：

208 \* @ 返回值：无

209 \/

**210 static void KEY_Task(void\* parameter)**

**211 {**

**212 /\* 任务都是一个无限循环，不能返回 \*/**

**213 while (1) {**

**214 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**215 printf ( "KEY1被按下\n" );**

**216 /\* 原型:BaseType_t xTaskNotify( TaskHandle_t xTaskToNotify,**

**217 uint32_t ulValue,**

**218 eNotifyAction eAction );**

**219 \* eNoAction = 0，通知任务而不更新其通知值。**

**220 \* eSetBits，设置任务通知值中的位。**

**221 \* eIncrement，增加任务的通知值。**

**222 \* eSetvaluewithoverwrite，覆盖当前通知**

**223 \* eSetValueWithoutoverwrite 不覆盖当前通知**

**224 \**

**225 \* pdFAIL：当参数eAction设置为eSetValueWithoutOverwrite的时候，**

**226 \* 如果任务通知值没有更新成功就返回pdFAIL。**

**227 \* pdPASS: eAction 设置为其他选项的时候统一返回pdPASS。**

**228 \*/**

**229 /\* 触发一个事件1 \*/**

**230 xTaskNotify((TaskHandle_t)LED_Task_Handle,//接收任务通知的任务句柄**

**231 (uint32_t)KEY1_EVENT,//要触发的事件**

**232 (eNotifyAction)eSetBits); //设置任务通知值中的位**

**233**

**234 }**

**235**

**236 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**237 printf ( "KEY2被按下\n" );**

**238 /\* 触发一个事件2 \*/**

**239 xTaskNotify((TaskHandle_t )LED_Task_Handle,//接收任务通知的任务句柄**

**240 (uint32_t )KEY2_EVENT,//要触发的事件**

**241 (eNotifyAction)eSetBits);//设置任务通知值中的位**

**242 }**

**243 vTaskDelay(20); //每20ms扫描一次**

**244 }**

**245 }**

246

247 /\*

248 \* @ 函数名： BSP_Init

249 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

250 \* @ 参数：

251 \* @ 返回值：无

252 \/

253 static void BSP_Init(void)

254 {

255 /\*

256 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

257 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

258 \* 都统一用这个优先级分组，千万不要再分组，切忌。

259 \*/

260 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

261

262 /\* LED 初始化 \*/

263 LED_GPIO_Config();

264

265 /\* 串口初始化 \*/

266 USART_Config();

267

268 /\* 按键初始化 \*/

269 Key_GPIO_Config();

270

271 }

272

273 /END OF FILE/

任务通知实验现象
~~~~~~~~

任务通知代替消息队列实验现象
^^^^^^^^^^^^^^

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，按下开发版的KEY1按键发
送消息1，按下KEY2按键发送消息2；我们按下KEY1试试，在串口调试助手中可以看到接收到消息1，我们按下KEY2试试，在串口调试助手中可以看到接收到消息2，具体见图20‑1。

|taskno002|

图20‑1任务通知代替消息队列实验现象

任务通知代替二值信号量实验现象
^^^^^^^^^^^^^^^

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，它里面输出了信息表明任务正
在运行中，我们按下开发板的按键，串口打印任务运行的信息，表明两个任务同步成功，具体见图20‑2。

|taskno003|

图20‑2任务通知代替二值信号量实验现象

任务通知代替计数信号量实验现象
^^^^^^^^^^^^^^^

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，按下开发版的KEY1按键获
取信号量模拟停车，按下KEY2按键释放信号量模拟取车，因为是使用任务通知代替信号量，所以任务通知值默认为0，表当前车位为0；我们按下KEY1与KEY2试试，在串口调试助手中可以看到运行结果，具体见图20‑3。

|taskno004|

图20‑3任务通知代替计数信号量实验现象

任务通知代替事件组实验现象
^^^^^^^^^^^^^

程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，按下开发版的KEY1按键发送
事件通知1，按下KEY2按键发送事件通知2；我们按下KEY1与KEY2试试，在串口调试助手中可以看到运行结果，并且当事件1与事件2都发生的时候，开发板的LED会进行翻转，具体见图20‑4。

|taskno005|

图20‑4任务通知代替事件组实验现象

.. |taskno002| image:: media\taskno002.png
   :width: 5.43241in
   :height: 2.82567in
.. |taskno003| image:: media\taskno003.png
   :width: 5.45194in
   :height: 2.81169in
.. |taskno004| image:: media\taskno004.png
   :width: 5.66234in
   :height: 2.94728in
.. |taskno005| image:: media\taskno005.png
   :width: 5.24038in
   :height: 4.35821in
