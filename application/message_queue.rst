.. vim: syntax=rst

消息队列
============

同学们，回想一下，在我们裸机的编程中，我们是怎么样用全局的一个数组的呢？

消息队列的基本概念
~~~~~~~~~

队列又称消息队列，是一种常用于任务间通信的数据结构，队列可以在任务与任务间、中断和任务间传递信息，实现了任务接收来自其他任务或中断的不固定长度的消息，任务能够从队列里面读取消息，当队列中的消息是空时，读取消息的任务将被阻塞，用户还可以指定阻塞的任务时间xTicksToWait，在这段时间中，如果队列
为空，该任务将保持阻塞状态以等待队列数据有效。当队列中有新消息时，被阻塞的任务会被唤醒并处理新消息；当等待的时间超过了指定的阻塞时间，即使队列中尚无有效数据，任务也会自动从阻塞态转为就绪态。消息队列是一种异步的通信方式。

通过消息队列服务，任务或中断服务例程可以将一条或多条消息放入消息队列中。同样，一个或多个任务可以从消息队列中获得消息。当有多个消息发送到消息队列时，通常是将先进入消息队列的消息先传给任务，也就是说，任务先得到的是最先进入消息队列的消息，即先进先出原则（FIFO），但是也支持后进先出原则（LIFO）。

FreeRTOS中使用队列数据结构实现任务异步通信工作，具有如下特性：

-  消息支持先进先出方式排队，支持异步读写工作方式。

-  读写队列均支持超时机制。

-  消息支持后进先出方式排队，往队首发送消息（LIFO）。

-  可以允许不同长度（不超过队列节点最大值）的任意类型消息。

-  一个任务能够从任意一个消息队列接收和发送消息。

-  多个任务能够从同一个消息队列接收和发送消息。

-  当队列使用结束后，可以通过删除队列函数进行删除。

消息队列的运作机制
~~~~~~~~~

创建消息队列时FreeRTOS会先给消息队列分配一块内存空间，这块内存的大小等于消息队列控制块大小加上（单个消息空间大小与消息队列长度的乘积），接着再初始化消息队列，此时消息队列为空。FreeRTOS的消息队列控制块由多个元素组成，当消息队列被创建时，系统会为控制块分配对应的内存空间，用于保存消息队
列的一些信息如消息的存储位置，头指针pcHead、尾指针pcTail、消息大小uxItemSize以及队列长度uxLength等。同时每个消息队列都与消息空间在同一段连续的内存空间中，在创建成功的时候，这些内存就被占用了，只有删除了消息队列的时候，这段内存才会被释放掉，创建成功的时候就已经分配好每个
消息空间与消息队列的容量，无法更改，每个消息空间可以存放不大于消息大小uxItemSize的任意类型的数据，所有消息队列中的消息空间总数即是消息队列的长度，这个长度可在消息队列创建时指定。

任务或者中断服务程序都可以给消息队列发送消息，当发送消息时，如果队列未满或者允许覆盖入队，FreeRTOS会将消息拷贝到消息队列队尾，否则，会根据用户指定的阻塞超时时间进行阻塞，在这段时间中，如果队列一直不允许入队，该任务将保持阻塞状态以等待队列允许入队。当其他任务从其等待的队列中读取入了数据（队列
未满），该任务将自动由阻塞态转移为就绪态。当等待的时间超过了指定的阻塞时间，即使队列中还不允许入队，任务也会自动从阻塞态转移为就绪态，此时发送消息的任务或者中断程序会收到一个错误码errQUEUE_FULL。

发送紧急消息的过程与发送消息几乎一样，唯一的不同是，当发送紧急消息时，发送的位置是消息队列队头而非队尾，这样，接收者就能够优先接收到紧急消息，从而及时进行消息处理。

当某个任务试图读一个队列时，其可以指定一个阻塞超时时间。在这段时间中，如果队列为空，该任务将保持阻塞状态以等待队列数据有效。当其他任务或中断服务程序往其等待的队列中写入了数据，该任务将自动由阻塞态转移为就绪态。当等待的时间超过了指定的阻塞时间，即使队列中尚无有效数据，任务也会自动从阻塞态转移为就绪态
。

当消息队列不再被使用时，应该删除它以释放系统资源，一旦操作完成，消息队列将被永久性的删除。

消息队列的运作过程具体见图15‑1。

|messag002|

图15‑1消息队列运作过程

消息队列的阻塞机制
~~~~~~~~~

我们使用的消息队列一般不是属于某个任务的队列，在很多时候，我们创建的队列，是每个任务都可以去对他进行读写操作的，但是为了保护每个任务对它进行读写操作的过程，我们必须有阻塞机制，在某个任务对它读写操作的时候，必须保证该任务能正常完成读写操作，而不受后来的任务干扰，凡事都有先来后到嘛！

那么，如何实现这个先来后到的机制呢，很简单，因为FreeRTOS已经为我们做好了，我们直接使用就好了，每个对消息队列读写的函数，都有这种机制，我称之为阻塞机制。假设有一个任务A对某个队列进行读操作的时候（也就是我们所说的出队），发现它没有消息，那么此时任务A有3个选择：第一个选择，任务A扭头就走，既
然队列没有消息，那我也不等了，干其他事情去，这样子任务A不会进入阻塞态；第二个选择，任务A还是在这里等等吧，可能过一会队列就有消息，此时任务A会进入阻塞状态，在等待着消息的道来，而任务A的等待时间就由我们自己定义，比如设置1000个系统时钟节拍tick的等待，在这1000个tick到来之前任务A都是
处于阻塞态，当阻塞的这段时间任务A等到了队列的消息，那么任务A就会从阻塞态变成就绪态，如果此时任务A比当前运行的任务优先级还高，那么，任务A就会得到消息并且运行；假如1000个tick都过去了，队列还没消息，那任务A就不等了，从阻塞态中唤醒，返回一个没等到消息的错误代码，然后继续执行任务A的其他代码
；第三个选择，任务A死等，不等到消息就不走了，这样子任务A就会进入阻塞态，直到完成读取队列的消息。

而在发送消息操作的时候，为了保护数据，当且仅当队列允许入队的时候，发送者才能成功发送消息；队列中无可用消息空间时，说明消息队列已满，此时，系统会根据用户指定的阻塞超时时间将任务阻塞，在指定的超时时间内如果还不能完成入队操作，发送消息的任务或者中断服务程序会收到一个错误码errQUEUE_FULL，然
后解除阻塞状态；当然，只有在任务中发送消息才允许进行阻塞状态，而在中断中发送消息不允许带有阻塞机制的，需要调用在中断中发送消息的API函数接口，因为发送消息的上下文环境是在中断中，不允许有阻塞的情况。

假如有多个任务阻塞在一个消息队列中，那么这些阻塞的任务将按照任务优先级进行排序，优先级高的任务将优先获得队列的访问权。

消息队列的应用场景
~~~~~~~~~

消息队列可以应用于发送不定长消息的场合，包括任务与任务间的消息交换，队列是FreeRTOS主要的任务间通信方式，可以在任务与任务间、中断和任务间传送信息，发送到队列的消息是通过拷贝方式实现的，这意味着队列存储的数据是原数据，而不是原数据的引用。

消息队列控制块
~~~~~~~

FreeRTOS的消息队列控制块由多个元素组成，当消息队列被创建时，系统会为控制块分配对应的内存空间，用于保存消息队列的一些信息如消息的存储位置，头指针pcHead、尾指针pcTail、消息大小uxItemSize以及队列长度uxLength，以及当前队列消息个数uxMessagesWaiting等
，具体见代码清单15‑1。

代码清单15‑1消息队列控制块

1 typedefstruct QueueDefinition {

2 int8_t \*pcHead; **(1)**

3 int8_t \*pcTail; **(2)**

4 int8_t \*pcWriteTo; **(3)**

5

6 union {

7 int8_t \*pcReadFrom; **(4)**

8 UBaseType_t uxRecursiveCallCount; **(5)**

9 } u;

10

11 List_t xTasksWaitingToSend; **(6)**

12 List_t xTasksWaitingToReceive; **(7)**

13

14 volatile UBaseType_t uxMessagesWaiting; **(8)**

15 UBaseType_t uxLength; **(9)**

16 UBaseType_t uxItemSize; **(10)**

17

18 volatileint8_t cRxLock; **(11)**

19 volatileint8_t cTxLock; **(12)**

20

21 #if( ( configSUPPORT_STATIC_ALLOCATION == 1 )

22 && ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) )

23 uint8_t ucStaticallyAllocated;

24 #endif

25

26 #if ( configUSE_QUEUE_SETS == 1 )

27 struct QueueDefinition \*pxQueueSetContainer;

28 #endif

29

30 #if ( configUSE_TRACE_FACILITY == 1 )

31 UBaseType_t uxQueueNumber;

32 uint8_t ucQueueType;

33 #endif

34

35 } xQUEUE;

36

37 typedef xQUEUE Queue_t;

代码清单15‑1\ **(1)**\ ：pcHead指向队列消息存储区起始位置，即第一个消息空间。

代码清单15‑1\ **(2)**\ ：pcTail指向队列消息存储区结束位置地址。

代码清单15‑1\ **(3)**\ ：pcWriteTo指向队列消息存储区下一个可用消息空间。

代码清单15‑1\ **(4)**\
：pcReadFrom与uxRecursiveCallCount是一对互斥变量，使用联合体用来确保两个互斥的结构体成员不会同时出现。当结构体用于队列时，pcReadFrom指向出队消息空间的最后一个，见文知义，就是读取消息时候是从pcReadFrom指向的空间读取消息内容。

代码清单15‑1\ **(5)**\ ：当结构体用于互斥量时，uxRecursiveCallCount用于计数，记录递归互斥量被“调用”的次数。

代码清单15‑1\ **(6)**\ ：xTasksWaitingToSend是一个发送消息阻塞列表，用于保存阻塞在此队列的任务，任务按照优先级进行排序，由于队列已满，想要发送消息的任务无法发送消息。

代码清单15‑1\ **(7)**\ ：xTasksWaitingToReceive是一个获取消息阻塞列表，用于保存阻塞在此队列的任务，任务按照优先级进行排序，由于队列是空的，想要获取消息的任务无法获取到消息。

代码清单15‑1\ **(8)**\ ：uxMessagesWaiting用于记录当前消息队列的消息个数，如果消息队列被用于信号量的时候，这个值就表示有效信号量个数。

代码清单15‑1\ **(9)**\ ：uxLength表示队列的长度，也就是能存放多少消息。

代码清单15‑1\ **(10)**\ ：uxItemSize表示单个消息的大小。

代码清单15‑1\ **(11)**\ ：队列上锁后，储存从队列收到的列表项数目，也就是出队的数量，如果队列没有上锁，设置为queueUNLOCKED。

代码清单15‑1\ **(12)**\ ：队列上锁后，储存发送到队列的列表项数目，也就是入队的数量，如果队列没有上锁，设置为queueUNLOCKED。

这两个成员变量为queueUNLOCKED时，表示队列未上锁；当这两个成员变量为queueLOCKED_UNMODIFIED时，表示队列上锁。

消息队列常用函数讲解
~~~~~~~~~~

使用队列模块的典型流程如下：

-  创建消息队列。

-  写队列操作。

-  读队列操作。

-  删除队列。

消息队列创建函数xQueueCreate()
^^^^^^^^^^^^^^^^^^^^^^

xQueueCreate()用于创建一个新的队列并返回可用于访问这个队列的队列句柄。队列句柄其实就是一个指向队列数据结构类型的指针。

队列就是一个数据结构，用于任务间的数据的传递。每创建一个新的队列都需要为其分配RAM，一部分用于存储队列的状态，剩下的作为队列消息的存储区域。使用xQueueCreate()创建队列时，使用的是动态内存分配，所以要想使用该函数必须在FreeRTOSConfig.h中把\
`configSUPPORT_DYNAMIC_ALLOCATION <http://www.freertos.org/a00110.html#configSUPPORT_DYNAMIC_ALLOCATION>`__\
定义为1来使能，这是个用于使能动态内存分配的宏，通常情况下，在FreeRTOS中，凡是创建任务，队列，信号量和互斥量等内核对象都需要使用动态内存分配，所以这个宏默认在FreeRTOS.h头文件中已经使能（即定义为1）。如果想使用静态内存，则可以使用\ `xQueueCreateStatic() <h
ttp://www.freertos.org/xQueueCreateStatic.html>`__ 函数来创建一个队列。使用静态创建消息队列函数创建队列时需要的形参更多，需要的内存由编译的时候预先分配好，一般很少使用这种方法。xQueueCreate()函数原型具体见代码清单15‑2加粗部分，使用
说明具体见表15‑1。

代码清单15‑2xQueueCreate()函数原型

1 #if( configSUPPORT_DYNAMIC_ALLOCATION == 1 )

**2 #define xQueueCreate( uxQueueLength, uxItemSize ) \\**

**3 xQueueGenericCreate( ( uxQueueLength ), ( uxItemSize ), ( queueQUEUE_TYPE_BASE ) )**

4 #endif

表15‑1xQueueCreate()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Qu
     - ueHandle_t            | xQueueCreate( UBaseType_t uxQueueLength,  UBaseType_t uxItemSize );
     - |

   * - **功能**     |
     - 于创建一个新的队列。   |
     - |

   * - **参数**     |
     - xQueueLength            |
     - 列能够存储的最大消     | 息单元数目，即队列长度。 |

   * -
     - uxItemSize
     - 队列中消息单             | 元的大小，以字节为单位。 |

   * - **返回值**   | 如
     - 创建成功则返回一个队 | 列句柄，用于访问创建的队 | 列。如果创建不成功则返回 | NULL，可能原因是创建队列 | 需要的RAM无法分配成功。  |
     - |
          |
          |
            |
            |


从函数原型中，我们可以看到，创建队列真正使用的函数是xQueueGenericCreate()，消息队列创建函数，顾名思义，就是创建一个队列，与任务一样，都是需要先创建才能使用的东西，FreeRTOS肯定不知道我们需要什么样的队列，比如队列的长度，消息的大小这些信息都是需要我们自己定义的，FreeR
TOS提供给我们这个创建函数，爱怎么搞都是我们自己来实现，下面来看看xQueueGenericCreate()函数源码，具体见代码清单15‑3。

代码清单15‑3xQueueGenericCreate()函数源码

1 /*-----------------------------------------------------------*/

2 #if( configSUPPORT_DYNAMIC_ALLOCATION == 1 )

3

4 QueueHandle_t xQueueGenericCreate( const UBaseType_t uxQueueLength,

5 const UBaseType_t uxItemSize,

6 const uint8_t ucQueueType )

7 {

8 Queue_t \*pxNewQueue;

9 size_t xQueueSizeInBytes;

10 uint8_t \*pucQueueStorage;

11

12 configASSERT( uxQueueLength > ( UBaseType_t ) 0 );

13

14 if ( uxItemSize == ( UBaseType_t ) 0 ) {

15 /\* 消息空间大小为0*/

16 xQueueSizeInBytes = ( size_t ) 0; **(1)**

17 } else {

18 /\* 分配足够消息存储空间，空间的大小为队列长度*单个消息大小 \*/

19 xQueueSizeInBytes = ( size_t ) ( uxQueueLength \* uxItemSize );\ **(2)**

20 }

21 /\* 向系统申请内存，内存大小为消息队列控制块大小+消息存储空间大小 \*/

22 pxNewQueue=(Queue_t*)pvPortMalloc(sizeof(Queue_t)+xQueueSizeInBytes);\ **(3)**

23

24 if ( pxNewQueue != NULL ) {

25 /\* 计算出消息存储空间的起始地址 \*/

26 pucQueueStorage = ( ( uint8_t \* ) pxNewQueue ) + sizeof( Queue_t );\ **(4)**

27

28 #if( configSUPPORT_STATIC_ALLOCATION == 1 )

29 {

30

31 pxNewQueue->ucStaticallyAllocated = pdFALSE;

32 }

33 #endif

34

35 prvInitialiseNewQueue( uxQueueLength, **(5)**

36 uxItemSize,

37 pucQueueStorage,

38 ucQueueType,

39 pxNewQueue );

40 }

41

42 return pxNewQueue;

43 }

44

45 #endif

46 /*-----------------------------------------------------------*/

代码清单15‑3\ **(1)**\ ：如果uxItemSize为0，也就是单个消息空间大小为0，这样子就不需要申请内存了，那么xQueueSizeInBytes也设置为0即可，设置为0是可以的，用作信号量的时候这个就可以设置为0。

代码清单15‑3\ **(2)**\ ：uxItemSize并不是为0，那么需要分配足够存储消息的空间，内存的大小为队列长度*单个消息大小。

代码清单15‑3\ **(3)**\ ：FreeRTOS调用pvPortMalloc()函数向系统申请内存空间，内存大小为消息队列控制块大小加上消息存储空间大小，因为这段内存空间是需要保证连续的，具体见图15‑2。

|messag003|

图15‑2消息队列的内存空间示意图

代码清单15‑3\ **(4)**\ ：计算出消息存储内存空间的起始地址，因为\ **(3)**\ 步骤中申请的内存是包含了消息队列控制块的内存空间，但是我们存储消息的内存空间在消息队列控制块后面。

代码清单15‑3\ **(5)**\ ：调用prvInitialiseNewQueue()函数将消息队列进行初始化。其实xQueueGenericCreate()主要是用于分配消息队列内存的，消息队列初始化函数源码具体见代码清单15‑4。

代码清单15‑4prvInitialiseNewQueue()函数源码

1 /*-----------------------------------------------------------*/

2 static void prvInitialiseNewQueue( const UBaseType_t uxQueueLength,\ **(1)**

3 const UBaseType_t uxItemSize,\ **(2)**

4 uint8_t \*pucQueueStorage, **(3)**

5 const uint8_t ucQueueType, **(4)**

6 Queue_t \*pxNewQueue ) **(5)**

7 {

8 ( void ) ucQueueType;

9

10 if ( uxItemSize == ( UBaseType_t ) 0 ) {

11 /\* 没有为消息存储分配内存,但是pcHead指针不能设置为NULL,

12 因为队列用作互斥量时,pcHead要设置成NULL。

13 这里只是将pcHead指向一个已知的区域 \*/

14 pxNewQueue->pcHead = ( int8_t \* ) pxNewQueue; **(6)**

15 } else {

16 /\* 设置pcHead指向存储消息的起始地址 \*/

17 pxNewQueue->pcHead = ( int8_t \* ) pucQueueStorage; **(7)**

18 }

19

20 /\* 初始化消息队列控制块的其他成员 \*/

21 pxNewQueue->uxLength = uxQueueLength; **(8)**

22 pxNewQueue->uxItemSize = uxItemSize;

23 /\* 重置消息队列 \*/

24 ( void ) xQueueGenericReset( pxNewQueue, pdTRUE ); **(9)**

25

26 #if ( configUSE_TRACE_FACILITY == 1 )

27 {

28 pxNewQueue->ucQueueType = ucQueueType;

29 }

30 #endif

31

32 #if( configUSE_QUEUE_SETS == 1 )

33 {

34 pxNewQueue->pxQueueSetContainer = NULL;

35 }

36 #endif

37

38 traceQUEUE_CREATE( pxNewQueue );

39 }

40 /*-----------------------------------------------------------*/

代码清单15‑4\ **(1)**\ ：消息队列长度。

代码清单15‑4\ **(2)**\ ：单个消息大小。

代码清单15‑4\ **(3)**\ ：存储消息起始地址。

代码清单15‑4\ **(4)**\ ：消息队列类型：

-  queueQUEUE_TYPE_BASE：表示队列。

-  queueQUEUE_TYPE_SET：表示队列集合。

-  queueQUEUE_TYPE_MUTEX：表示互斥量。

-  queueQUEUE_TYPE_COUNTING_SEMAPHORE：表示计数信号量。

-  queueQUEUE_TYPE_BINARY_SEMAPHORE：表示二进制信号量。

-  queueQUEUE_TYPE_RECURSIVE_MUTEX ：表示递归互斥量。

代码清单15‑4\ **(5)**\ ：消息队列控制块。

代码清单15‑4\ **(6)**\ ：如果没有为消息队列分配存储消息的内存空间，而且pcHead指针不能设置为NULL，因为队列用作互斥量时，pcHead要设置成NULL，这里只能将pcHead指向一个已知的区域，指向消息队列控制块pxNewQueue。

代码清单15‑4\ **(7)**\ ：如果分配了存储消息的内存空间，则设置pcHead指向存储消息的起始地址pucQueueStorage。

代码清单15‑4\ **(8)**\ ：初始化消息队列控制块的其他成员，消息队列的长度与消息的大小。

代码清单15‑4\ **(9)**\ ：重置消息队列，在消息队列初始化的时候，需要重置一下相关参数，具体见代码清单15‑5。

代码清单15‑5重置消息队列xQueueGenericReset()源码

1 /*-----------------------------------------------------------*/

2 BaseType_t xQueueGenericReset( QueueHandle_t xQueue,

3 BaseType_t xNewQueue )

4 {

5 Queue_t \* const pxQueue = ( Queue_t \* ) xQueue;

6

7 configASSERT( pxQueue );

8

9 taskENTER_CRITICAL(); **(1)**

10 {

11 pxQueue->pcTail = pxQueue->pcHead +

12 ( pxQueue->uxLength \* pxQueue->uxItemSize ); **(2)**

13 pxQueue->uxMessagesWaiting = ( UBaseType_t ) 0U; **(3)**

14 pxQueue->pcWriteTo = pxQueue->pcHead; **(4)**

15 pxQueue->u.pcReadFrom = pxQueue->pcHead +

16 (( pxQueue->uxLength - ( UBaseType_t ) 1U ) \* pxQueue->uxItemSize );\ **(5)**

17 pxQueue->cRxLock = queueUNLOCKED; **(6)**

18 pxQueue->cTxLock = queueUNLOCKED;

19

20 if ( xNewQueue == pdFALSE ) { **(7)**

21 if ( listLIST_IS_EMPTY

22 ( &( pxQueue->xTasksWaitingToSend ) ) == pdFALSE ) {

23 if ( xTaskRemoveFromEventList

24 ( &( pxQueue->xTasksWaitingToSend ) ) != pdFALSE ) {

25 queueYIELD_IF_USING_PREEMPTION();

26 } else {

27 mtCOVERAGE_TEST_MARKER();

28 }

29 } else {

30 mtCOVERAGE_TEST_MARKER();

31 }

32 } else { **(8)**

33 vListInitialise( &( pxQueue->xTasksWaitingToSend ) );

34 vListInitialise( &( pxQueue->xTasksWaitingToReceive ) );

35 }

36 }

37 taskEXIT_CRITICAL(); **(9)**

38

39 return pdPASS;

40 }

41 /*-----------------------------------------------------------*/

代码清单15‑5\ **(1)**\ ：进入临界段。

代码清单15‑5\ **(2)**\ ：重置消息队列的成员变量，pcTail指向存储消息内存空间的结束地址。

代码清单15‑5\ **(3)**\ ：当前消息队列中的消息个数uxMessagesWaiting为0。

代码清单15‑5\ **(4)**\ ：pcWriteTo指向队列消息存储区下一个可用消息空间，因为是重置消息队列，就指向消息队列的第一个消息空间，也就是pcHead指向的空间。

代码清单15‑5\ **(5)**\ ：pcReadFrom指向消息队列最后一个消息空间。

代码清单15‑5\ **(6)**\ ：消息队列没有上锁，设置为queueUNLOCKED。

代码清单15‑5\ **(7)**\ ：如果不是新建一个消息队列，那么之前的消息队列可能阻塞了一些任务，需要将其解除阻塞。如果有发送消息任务被阻塞，那么需要将它恢复，而如果任务是因为读取消息而阻塞，那么重置之后的消息队列也是空的，则无需被恢复。

代码清单15‑5\ **(8)**\ ：如果是新创建一个消息队列，则需要将xTasksWaitingToSend列表与xTasksWaitingToReceive列表初始化，列表的初始化在前面的章节已经讲解了，具体见4.2 小节。

代码清单15‑5\ **(9)**\ ：退出临界段。

至此，消息队列的创建就讲解完毕，创建完成的消息队列示意图具体见图15‑3。

|messag004|

图15‑3消息队列创建完成示意图

在创建消息队列的时候，是需要用户自己定义消息队列的句柄的，但是注意了，定义了队列的句柄并不等于创建了队列，创建队列必须是调用消息队列创建函数进行创建（可以是静态也可以是动态创建），否则，以后根据队列句柄使用消息队列的其他函数的时候会发生错误，创建完成会返回消息队列的句柄，用户通过句柄就可使用消息队列
进行发送与读取消息队列的操作，如果返回的是NULL则表示创建失败，消息队列创建函数xQueueCreate()使用实例具体见代码清单15‑6加粗部分。

代码清单15‑6xQueueCreate()实例

1 QueueHandle_t Test_Queue =NULL;

2

3 #define QUEUE_LEN 4/\* 队列的长度，最大可包含多少个消息 \*/

4 #define QUEUE_SIZE 4/\* 队列中每个消息大小（字节） \*/

5

6 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

7

8 taskENTER_CRITICAL(); //进入临界区

9

**10 /\* 创建Test_Queue \*/**

**11 Test_Queue = xQueueCreate((UBaseType_t ) QUEUE_LEN,/\* 消息队列的长度 \*/**

**12 (UBaseType_t ) QUEUE_SIZE);/\* 消息的大小 \*/**

**13 if (NULL != Test_Queue)**

**14 printf("创建Test_Queue消息队列成功!\r\n");**

15

16 taskEXIT_CRITICAL(); //退出临界区

消息队列静态创建函数xQueueCreateStatic()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

xQueueCreateStatic()用于创建一个新的队列并返回可用于访问这个队列的队列句柄。队列句柄其实就是一个指向队列数据结构类型的指针。

队列就是一个数据结构，用于任务间的数据的传递。每创建一个新的队列都需要为其分配RAM，一部分用于存储队列的状态，剩下的作为队列的存储区。使用xQueueCreateStatic()创建队列时，使用的是静态内存分配，所以要想使用该函数必须在FreeRTOSConfig.h中把configSUPPORT
_STATIC_ALLOCATION定义为1来使能。这是个用于使能静态内存分配的宏，需要的内存在程序编译的时候分配好，由用户自己定义，其实创建过程与xQueueCreate()都是差不多的，我们暂不深入讲解。
xQueueCreateStatic()函数的具体说明见表15‑2，使用实例具体见代码清单15‑7加粗部分。

表15‑2xQueueCreateStatic()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Qu
     - ueHandle_t            | xQueue CreateStatic(UBaseType_t uxQueueLength,  UBaseType_t uxItemSize,  uint8_t \*pucQueueStorageBuffer,  StaticQueue_t
       \*pxQueueBuffer );
     - |

   * - **功能**     |
     - 于创建一个新的队列。   |
     - |

   * - **参数**     |
     - xQueueLength            |
     - 列能够存储的最         | 大单元数目，即队列深度。 |

   * -
     - uxItemSize
     - 队列中数据单             | 元的长度，以字节为单位。 |

   * -
     - pucQueueStorageBuffer
     - 指针，指向一个uin        | t8_t类型的数组，数组的大 | 小至少有uxQueueLength\*  | uxItemSize个字节。当ux   | ItemSize为0时，pucQueueS | torageBuffer可以为NULL。 |

   * -
     - pxQueueBuffer
     - 指针，指向StaticQ        | ueue_t类型的变量，该变量 | 用于存储队列的数据结构。 |

   * - **返回值**   | 如
     - 创建成功则返回一个队 | 列句柄，用于访问创建的队 | 列。如果创建不成功则返回 | NULL，可能原因是创建队列 | 需要的RAM无法分配成功。  |
     - |
          |
          |
            |
            |


代码清单15‑7xQueueCreateStatic()函数使用实例

1 /\* 创建一个可以最多可以存储10个64位变量的队列 \*/

2 #define QUEUE_LENGTH 10

3 #define ITEM_SIZE sizeof( uint64_t )

4

5 /\* 该变量用于存储队列的数据结构 \*/

6 static StaticQueue_t xStaticQueue;

7

8 /\* 该数组作为队列的存储区域，大小至少有uxQueueLength \* uxItemSize个字节 \*/

**9 uint8_t ucQueueStorageArea[ QUEUE_LENGTH \* ITEM_SIZE ];**

10

11 void vATask( void \*pvParameters )

12 {

13 QueueHandle_t xQueue;

14

**15 /\* 创建一个队列 \*/**

**16 xQueue = xQueueCreateStatic( QUEUE_LENGTH, /\* 队列深度 \*/**

**17 ITEM_SIZE, /\* 队列数据单元的单位 \*/**

**18 ucQueueStorageArea,/\* 队列的存储区域 \*/**

**19 &xStaticQueue ); /\* 队列的数据结构 \*/**

20 /\* 剩下的其他代码 \*/

21 }

消息队列删除函数vQueueDelete()
^^^^^^^^^^^^^^^^^^^^^^

队列删除函数是根据消息队列句柄直接删除的，删除之后这个消息队列的所有信息都会被系统回收清空，而且不能再次使用这个消息队列了，但是需要注意的是，如果某个消息队列没有被创建，那也是无法被删除的，动脑子想想都知道，没创建的东西就不存在，怎么可能被删除。xQueue是vQueueDelete()函数的形参，
是消息队列句柄，表示的是要删除哪个想队列，其函数源码具体见代码清单15‑8。

代码清单15‑8消息队列删除函数vQueueDelete()源码（已省略暂时无用部分）

1 void vQueueDelete( QueueHandle_t xQueue )

2 {

3 Queue_t \* const pxQueue = ( Queue_t \* ) xQueue;

4

5 /\* 断言 \*/

6 configASSERT( pxQueue ); **(1)**

7 traceQUEUE_DELETE( pxQueue );

8

9 #if ( configQUEUE_REGISTRY_SIZE > 0 )

10 {

11 /\* 将消息队列从注册表中删除，我们目前没有添加到注册表中，暂时不用理会 \*/

12 vQueueUnregisterQueue( pxQueue ); **(2)**

13 }

14 #endif

15

16 #if( ( configSUPPORT_DYNAMIC_ALLOCATION == 1 )

17 && ( configSUPPORT_STATIC_ALLOCATION == 0 ) ) {

18 /\* 因为用的消息队列是动态分配内存的，所以需要调用

19 vPortFree来释放消息队列的内存 \*/

20 vPortFree( pxQueue ); **(3)**

21 }

22 }

代码清单15‑8\ **(1)**\ ：对传入的消息队列句柄进行检查，如果消息队列是有效的才允许进行删除操作。

代码清单15‑8\ **(2)**\ ：将消息队列从注册表中删除，我们目前没有添加到注册表中，暂时不用理会。

代码清单15‑8\ **(3)**\ ：因为用的消息队列是动态分配内存的，所以需要调用vPortFree()函数来释放消息队列的内存。

消息队列删除函数vQueueDelete()的使用也是很简单的，只需传入要删除的消息队列的句柄即可，调用函数时，系统将删除这个消息队列。需要注意的是调用删除消息队列函数前，系统应存在xQueueCreate()或xQueueCreateStatic()函数创建的消息队列。此外vQueueDelete
()也可用于删除信号量。如果删除消息队列时，有任务正在等待消息，则不应该进行删除操作（官方说的是不允许进行删除操作，但是源码并没有禁止删除的操作，使用的时候注意一下就行了），删除消息队列的实例具体见代码清单15‑9加粗部分。

代码清单15‑9消息队列删除函数vQueueDelete()使用实例

1 #define QUEUE_LENGTH 5

2 #define QUEUE_ITEM_SIZE 4

3

4 int main( void )

5 {

6 QueueHandle_t xQueue;

7 /\* 创建消息队列 \*/

8 xQueue = xQueueCreate( QUEUE_LENGTH, QUEUE_ITEM_SIZE );

9

10 if ( xQueue == NULL ) {

11 /\* 消息队列创建失败 \*/

12 } else {

**13 /\* 删除已创建的消息队列 \*/**

**14 vQueueDelete( xQueue );**

15 }

16 }

向消息队列发送消息函数
^^^^^^^^^^^

任务或者中断服务程序都可以给消息队列发送消息，当发送消息时，如果队列未满或者允许覆盖入队，FreeRTOS会将消息拷贝到消息队列队尾，否则，会根据用户指定的阻塞超时时间进行阻塞，在这段时间中，如果队列一直不允许入队，该任务将保持阻塞状态以等待队列允许入队。当其他任务从其等待的队列中读取入了数据（队列
未满），该任务将自动由阻塞态转为就绪态。当任务等待的时间超过了指定的阻塞时间，即使队列中还不允许入队，任务也会自动从阻塞态转移为就绪态，此时发送消息的任务或者中断程序会收到一个错误码errQUEUE_FULL。

发送紧急消息的过程与发送消息几乎一样，唯一的不同是，当发送紧急消息时，发送的位置是消息队列队头而非队尾，这样，接收者就能够优先接收到紧急消息，从而及时进行消息处理。

其实消息队列发送函数有好几个，都是使用宏定义进行展开的，有些只能在任务调用，有些只能在中断中调用，具体见下面讲解。

xQueueSend()与xQueueSendToBack()
'''''''''''''''''''''''''''''''

代码清单15‑10 xQueueSend()函数原型

1 #define xQueueSend( xQueue, pvItemToQueue, xTicksToWait ) \\

2 xQueueGenericSend( ( xQueue ), ( pvItemToQueue ), \\

3 ( xTicksToWait ), queueSEND_TO_BACK )

代码清单15‑11xQueueSendToBack()函数原型

1 #define xQueueSendToBack( xQueue, pvItemToQueue, xTicksToWait ) \\

2 xQueueGenericSend( ( xQueue ), ( pvItemToQueue ), \\

3 ( xTicksToWait ), queueSEND_TO_BACK )

xQueueSend()是一个宏，宏展开是调用函数xQueueGenericSend()，这个函数在后面会详细讲解其实现过程。该宏是为了向后兼容没有包含xQueueSendToFront()和xQueueSendToBack() 这两个宏的FreeRTOS版本。xQueueSend()等同于xQue
ueSendToBack()。

xQueueSend()用于向队列尾部发送一个队列消息。消息以拷贝的形式入队，而不是以引用的形式。该函数绝对不能在中断服务程序里面被调用，中断中必须使用带有中断保护功能的xQueueSendFromISR()来代替。xQueueSend()函数的具体说明见表15‑3，应用实例具体见代码清单15‑12
加粗部分。

表15‑3xQueueSend()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xQueueSend(QueueHandle_t xQueue,  const void \* pvItemToQueue,  TickType_t xTicksToWait);
     - |

   * - **功能**     |
     - 于向队                 | 列尾部发送一个队列消息。 |
     - |

   * - **参数**     |
     - Queue                   |
     - 列句柄。               |

   * -
     - pvItemToQueue
     - 指针，指向要发           | 送到队列尾部的队列消息。 |

   * -
     - xTicksToWait
     - 队                       | 列满时，等待队列空闲的最 | 大超时时间。如果队列满并 | 且xTicksToWait被设置成0  | ，函数立刻返回。超时时间 | 的单位为系统节拍周期，常 | 量portTICK_PERIOD_MS用于 | 辅助计算真实的时间，单位 |
       为ms。如果INCLUDE_vTask  | Suspend设置成1，并且指定 | 延时为portMAX_DELAY将导  | 致任务挂起（没有超时）。 |

   * - **返回值**   | 消
     - | 发送成功成功返回pdTRUE， | 否则返回errQUEUE_FULL。  |
     - |

       |


代码清单15‑12xQueueSend()函数使用实例

1 static void Send_Task(void\* parameter)

2 {

3 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

4 uint32_t send_data1 = 1;

5 uint32_t send_data2 = 2;

6 while (1) {

7 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {

8 /\* K1 被按下 \*/

**9 printf("发送消息send_data1！\n");**

**10 xReturn = xQueueSend( Test_Queue, /\* 消息队列的句柄 \*/**

**11 &send_data1,/\* 发送的消息内容 \*/**

**12 0 ); /\* 等待时间 0 \*/**

**13 if (pdPASS == xReturn)**

**14 printf("消息send_data1发送成功!\n\n");**

15 }

16 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {

17 /\* K2 被按下 \*/

**18 printf("发送消息send_data2！\n");**

**19 xReturn = xQueueSend( Test_Queue, /\* 消息队列的句柄 \*/**

**20 &send_data2,/\* 发送的消息内容 \*/**

**21 0 ); /\* 等待时间 0 \*/**

**22 if (pdPASS == xReturn)**

**23 printf("消息send_data2发送成功!\n\n");**

24 }

25 vTaskDelay(20);/\* 延时20个tick \*/

26 }

27 }

xQueueSendFromISR()与xQueueSendToBackFromISR()
'''''''''''''''''''''''''''''''''''''''''''''

代码清单15‑13xQueueSendFromISR()函数原型

1 #define xQueueSendFromISR( xQueue, pvItemToQueue,\\

2 pxHigherPriorityTaskWoken ) \\

3 xQueueGenericSendFromISR( ( xQueue ), ( pvItemToQueue ), \\

4 ( pxHigherPriorityTaskWoken ), queueSEND_TO_BACK )

xQueueSendToBackFromISR等同于xQueueSendFromISR ()。

代码清单15‑14 xQueueSendToBackFromISR()函数原型

1 #define xQueueSendToBackFromISR(xQueue,pvItemToQueue,pxHigherPriorityTaskWoken) \\

2 xQueueGenericSendFromISR( ( xQueue ), ( pvItemToQueue ), \\

3 ( pxHigherPriorityTaskWoken ), queueSEND_TO_BACK )

xQueueSendFromISR()是一个宏，宏展开是调用函数xQueueGenericSendFromISR()。该宏是xQueueSend()的中断保护版本，用于在中断服务程序中向队列尾部发送一个队列消息，等价于xQueueSendToBackFromISR()。xQueueSendFromI
SR()函数具体说明见表15‑4，使用实例具体见代码清单15‑15加粗部分。

表15‑4xQueueSendFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xQueueS endFromISR(QueueHandle_t xQueue,  const void \*pvItemToQueue,  BaseType_t \*pxH igherPriorityTaskWoken);
     - |

   * - **功能**     |
     - 中断服务程序中用于     | 向队列尾部发送一个消息。 |
     - |
       |

   * - **参数**     |
     - Queue                   |
     - 列句柄。               |

   * -
     - pvItemToQueue
     - 指针，指向               | 要发送到队列尾部的消息。 |

   * -
     - p xHigherPriorityTaskWoken
     - 如                       | 果入队导致一个任务解锁， | 并且解锁的任务优先级高于 | 当前被中断的任务，则将*p | xHigherPriorityTaskWoken 设置成pdTRUE，然后在中断 | 退出前需要进行一次上下文 | 切换，去执行被唤醒的优先 |
       级更高的任务。从FreeRTOS | V7.3.0起，pxHigherPri    | orityTaskWoken作为一个可 | 选参数，可以设置为NULL。 |

   * - **返回值**   | 消
     - 发送成功返回pdTRUE， | 否则返回errQUEUE_FULL。  |
     - |
              |


代码清单15‑15xQueueSendFromISR()函数使用实例

1 void vBufferISR( void )

2 {

3 char cIn;

**4 BaseType_t xHigherPriorityTaskWoken;**

5

6 /\* 在ISR开始的时候，我们并没有唤醒任务 \*/

**7 xHigherPriorityTaskWoken = pdFALSE;**

8

9 /\* 直到缓冲区为空 \*/

10 do {

11 /\* 从缓冲区获取一个字节的数据 \*/

12 cIn = portINPUT_BYTE( RX_REGISTER_ADDRESS );

13

**14 /\* 发送这个数据 \*/**

**15 xQueueSendFromISR( xRxQueue, &cIn, &xHigherPriorityTaskWoken );**

16

17 } while ( portINPUT_BYTE( BUFFER_COUNT ) );

18

**19 /\* 这时候buffer已经为空，如果需要则进行上下文切换 \*/**

**20 if ( xHigherPriorityTaskWoken ) {**

**21 /\* 上下文切换，这是一个宏，不同的处理器，具体的方法不一样 \*/**

**22 taskYIELD_FROM_ISR ();**

**23 }**

24 }

xQueueSendToFront()
'''''''''''''''''''

代码清单15‑16xQueueSendToFront()函数原型

1 #define xQueueSendToFront( xQueue, pvItemToQueue, xTicksToWait ) \\

2 xQueueGenericSend( ( xQueue ), ( pvItemToQueue ), \\

3 ( xTicksToWait ), queueSEND_TO_FRONT )

xQueueSendToFron()是一个宏，宏展开也是调用函数xQueueGenericSend()。xQueueSendToFront()用于向队列队首发送一个消息。消息以拷贝的形式入队，而不是以引用的形式。该函数绝不能在中断服务程序里面被调用，而是必须使用带有中断保护功能的xQueueSend
ToFrontFromISR ()来代替。xQueueSendToFron()函数的具体说明见表15‑5，使用方式与xQueueSend()函数一致。

表15‑5xQueueSendToFron()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xQueueSendToFront( QueueHandle_t xQueue,  const void \* pvItemToQueue,  TickType_t xTicksToWait );
     - |

   * - **功能**     |
     - | 向队列队首发送一个消息。 |
     - |

   * - **参数**     |
     - Queue                   |
     - 列句柄。               |

   * -
     - pvItemToQueue
     - 指针，                   | 指向要发送到队首的消息。 |

   * -
     - xTicksToWait
     - 队列满                   | 时，等待队列空闲的最大超 | 时时间。如果队列满并且x  | TicksToWait被设置成0，函 | 数立刻返回。超时时间的单 | 位为系统节拍周期，常量po | rtTICK_PERIOD_MS用于辅助 | 计算真实的时间，单位为m  |
       s。如果INCLUDE_vTaskSusp | end设置成1，并且指定延时 | 为portMAX_DELAY将导致任  | 务无限阻塞（没有超时）。 |

   * - **返回值**   | 发
     - 消息成功返回pdTRUE， | 否则返回errQUEUE_FULL。  |
     - |
              |


xQueueSendToFrontFromISR()
''''''''''''''''''''''''''

代码清单15‑17 xQueueSendToFrontFromISR()函数原型

1 #define xQueueSendToFrontFromISR( xQueue,pvItemToQueue,pxHigherPriorityTaskWoken ) \\

2 xQueueGenericSendFromISR( ( xQueue ), ( pvItemToQueue ), \\

3 ( pxHigherPriorityTaskWoken ), queueSEND_TO_FRONT )

xQueueSendToFrontFromISR()是一个宏，宏展开是调用函数xQueueGenericSendFromISR()。该宏是xQueueSendToFront()的中断保护版本，用于在中断服务程序中向消息队列队首发送一个消息。xQueueSendToFromISR()函数具体说明见表1
5‑6，使用方式与xQueueSendFromISR()函数一致。

表15‑6xQueueSendToFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xQueueSendToFr ontFromISR(QueueHandle_t xQueue,  const void \*pvItemToQueue,  BaseType_t \*pxH igherPriorityTaskWoken);
     - |

   * - **功能**     |
     - 中断服务程序中向消     | 息队列队首发送一个消息。 |
     - |
       |

   * - **参数**     |
     - Queue                   |
     - 列句柄。               |

   * -
     - pvItemToQueue
     - 指针，                   | 指向要发送到队首的消息。 |

   * -
     - p xHigherPriorityTaskWoken
     - 如                       | 果入队导致一个任务解锁， | 并且解锁的任务优先级高于 | 当前被中断的任务，则将*p | xHigherPriorityTaskWoken 设置成pdTRUE，然后在中断 | 退出前需要进行一次上下文 | 切换，去执行被唤醒的优先 |
       级更高的任务。从FreeRTOS | V7.3.0起，pxHigherPri    | orityTaskWoken作为一个可 | 选参数，可以设置为NULL。 |

   * - **返回值**   | 队
     - | 列项投递成功返回pdTRUE， | 否则返回errQUEUE_FULL。  |
     - |


通用消息队列发送函数xQueueGenericSend()（任务）
'''''''''''''''''''''''''''''''''

上面看到的那些在任务中发送消息的函数都是xQueueGenericSend()展开的宏定义，真正起作用的就是xQueueGenericSend()函数，根据指定的参数不一样，发送消息的结果就不一样，下面一起看看任务级的通用消息队列发送函数的实现过程，具体见代码清单15‑18。

代码清单15‑18 xQueueGenericSend()\ **函数源码（已删减）**

1 /*-----------------------------------------------------------*/

2 BaseType_t xQueueGenericSend( QueueHandle_t xQueue, **(1)**

3 const void \* const pvItemToQueue, **(2)**

4 TickType_t xTicksToWait, **(3)**

5 const BaseType_t xCopyPosition ) **(4)**

6 {

7 BaseType_t xEntryTimeSet = pdFALSE, xYieldRequired;

8 TimeOut_t xTimeOut;

9 Queue_t \* const pxQueue = ( Queue_t \* ) xQueue;

10

11 /\* 已删除一些断言操作 \*/

12

13 for ( ;; ) {

14 taskENTER_CRITICAL(); **(5)**

15 {

16 /\* 队列未满 \*/

17 if ( ( pxQueue->uxMessagesWaiting < pxQueue->uxLength )

18 \|\| ( xCopyPosition == queueOVERWRITE ) ) { **(6)**

19 traceQUEUE_SEND( pxQueue );

20 xYieldRequired =

21 prvCopyDataToQueue( pxQueue, pvItemToQueue, xCopyPosition );\ **(7)**

22

23 /\* 已删除使用队列集部分代码 \*/

24 /\* 如果有任务在等待获取此消息队列 \*/

25 if ( listLIST_IS_EMPTY(&(pxQueue->xTasksWaitingToReceive))==pdFALSE){**(8)**

26 /\* 将任务从阻塞中恢复 \*/

27 if ( xTaskRemoveFromEventList(

28 &( pxQueue->xTasksWaitingToReceive ) )!=pdFALSE) {**(9)**

29 /\* 如果恢复的任务优先级比当前运行任务优先级还高，

30 那么需要进行一次任务切换 \*/

31 queueYIELD_IF_USING_PREEMPTION(); **(10)**

32 } else {

33 mtCOVERAGE_TEST_MARKER();

34 }

35 } else if ( xYieldRequired != pdFALSE ) {

36 /\* 如果没有等待的任务，拷贝成功也需要任务切换 \*/

37 queueYIELD_IF_USING_PREEMPTION(); **(11)**

38 } else {

39 mtCOVERAGE_TEST_MARKER();

40 }

41

42 taskEXIT_CRITICAL(); **(12)**

43 return pdPASS;

44 }

45 /\* 队列已满 \*/

46 else { **(13)**

47 if ( xTicksToWait == ( TickType_t ) 0 ) {

48 /\* 如果用户不指定阻塞超时时间，退出 \*/

49 taskEXIT_CRITICAL(); **(14)**

50 traceQUEUE_SEND_FAILED( pxQueue );

51 return errQUEUE_FULL;

52 } else if ( xEntryTimeSet == pdFALSE ) {

53 /\* 初始化阻塞超时结构体变量，初始化进入

54 阻塞的时间xTickCount和溢出次数xNumOfOverflows \*/

55 vTaskSetTimeOutState( &xTimeOut ); **(15)**

56 xEntryTimeSet = pdTRUE;

57 } else {

58 mtCOVERAGE_TEST_MARKER();

59 }

60 }

61 }

62 taskEXIT_CRITICAL(); **(16)**

63 /\* 挂起调度器 \*/

64 vTaskSuspendAll();

65 /\* 队列上锁 \*/

66 prvLockQueue( pxQueue );

67

68 /\* 检查超时时间是否已经过去了 \*/

69 if (xTaskCheckForTimeOut(&xTimeOut, &xTicksToWait)==pdFALSE){**(17)**

70 /\* 如果队列还是满的 \*/

71 if ( prvIsQueueFull( pxQueue ) != pdFALSE ) { **(18)**

72 traceBLOCKING_ON_QUEUE_SEND( pxQueue );

73 /\* 将当前任务添加到队列的等待发送列表中

74 以及阻塞延时列表，延时时间为用户指定的超时时间xTicksToWait \*/

75 vTaskPlaceOnEventList(

76 &( pxQueue->xTasksWaitingToSend ), xTicksToWait );\ **(19)**

77 /\* 队列解锁 \*/

78 prvUnlockQueue( pxQueue ); **(20)**

79

80 /\* 恢复调度器 \*/

81 if ( xTaskResumeAll() == pdFALSE ) {

82 portYIELD_WITHIN_API();

83 }

84 } else {

85 /\* 队列有空闲消息空间，允许入队 \*/

86 prvUnlockQueue( pxQueue ); **(21)**

87 ( void ) xTaskResumeAll();

88 }

89 } else {

90 /\* 超时时间已过，退出 \*/

91 prvUnlockQueue( pxQueue ); **(22)**

92 ( void ) xTaskResumeAll();

93

94 traceQUEUE_SEND_FAILED( pxQueue );

95 return errQUEUE_FULL;

96 }

97 }

98 }

99 /*-----------------------------------------------------------*/

代码清单15‑18\ **(1)**\ ：消息队列句柄。

代码清单15‑18\ **(2)**\ ：指针，指向要发送的消息。

代码清单15‑18\ **(3)**\ ：指定阻塞超时时间。

代码清单15‑18\ **(4)**\ ：发送数据到消息队列的位置，有以下3个选择，在queue.h中有定义，queueSEND_TO_BACK：发送到队尾；queueSEND_TO_FRONT：发送到队头；queueOVERWRITE：以覆盖的方式发送。

代码清单15‑18\ **(5)**\ ：进入临界段。

代码清单15‑18\ **(6)**\ ：判断队列是否已满，而如果是使用覆盖的方式发送数据，无论队列满或者没满，都可以发送。

代码清单15‑18\ **(7)**\ ：如果队列没满，可以调用prvCopyDataToQueue()函数将消息拷贝到消息队列中。

代码清单15‑18\ **(8)**\ ：消息拷贝完毕，那么就看看有没有任务在等待消息。

代码清单15‑18\ **(9)**\ ：如果有任务在等待获取此消息，就要将任务从阻塞中恢复，调用xTaskRemoveFromEventList()函数将等待的任务从队列的等待接收列表xTasksWaitingToReceive中删除，并且添加到就绪列表中。

代码清单15‑18\ **(10)**\ ：将任务从阻塞中恢复，如果恢复的任务优先级比当前运行任务的优先级高，那么需要进行一次任务切换。

代码清单15‑18\ **(11)**\ ：如果没有等待的任务，拷贝成功也需要进行一次任务切换。

代码清单15‑18\ **(12)**\ ：退出临界段。

代码清单15‑18\ **(13)**\ ：\ **(7)-(12)**\ 是队列未满的操作，如果队列已满，又会不一样的操作过程。

代码清单15‑18\ **(14)**\ ：如果用户不指定阻塞超时时间，则直接退出，不会发送消息。

代码清单15‑18\ **(15)**\ ：而如果用户指定了超时时间，系统就会初始化阻塞超时结构体变量，初始化进入阻塞的时间xTickCount和溢出次数xNumOfOverflows，为后面的阻塞任务做准备。

代码清单15‑18\ **(16)**\ ：因为前面进入了临界段，所以应先退出临界段，并且把调度器挂起，因为接下来的操作系统不允许其他任务访问队列，简单粗暴挂起调度器就不会进行任务切换，但是挂起调度器并不会禁止中断的发生，所以还需给队列上锁，因为系统不希望突然有中断操作这个队列的xTasksWait
ingToReceive列表和xTasksWaitingToSend列表。

代码清单15‑18\ **(17)**\ ：检查一下用户指定的超时时间是否已经过去了。如果没过则执行\ **(18)-(21)**\ 。

代码清单15‑18\ **(18)**\ ：如果队列还是满的，系统只能根据用户指定的超时时间来阻塞一下任务。

代码清单15‑18\ **(19)**\ ：当前任务添加到队列的等待发送列表中，以及阻塞延时列表，阻塞时间为用户指定时间xTicksToWait。

代码清单15‑18\ **(20)**\ ：队列解锁，恢复调度器，如果调度器挂起期间有任务解除阻塞，并且解除阻塞的任务优先级比当前任务高，就需要进行一次任务切换。。

代码清单15‑18\ **(21)**\ ：队列有空闲消息空间，允许入队，就重新发送消息。

代码清单15‑18\ **(22)**\ ：超时时间已过，返回一个errQUEUE_FULL错误代码，退出。

从前面的函数中我们就知道怎么使用消息队列发送消息了，这里就不在重复赘述。

从消息队列的入队操作我们可以看出：如果阻塞时间不为0，则任务会因为等待入队而进入阻塞，在将任务设置为阻塞的过程中，系统不希望有其他任务和中断操作这个队列的xTasksWaitingToReceive列表和xTasksWaitingToSend列表，因为可能引起其他任务解除阻塞，这可能会发生优先级翻转
。比如任务A的优先级低于当前任务，但是在当前任务进入阻塞的过程中，任务A却因为其他原因解除阻塞了，这显然是要绝对禁止的。因此FreeRTOS使用挂起调度器禁止其他任务操作队列，因为挂起调度器意味着任务不能切换并且不准调用可能引起任务切换的API函数。但挂起调度器并不会禁止中断，中断服务函数仍然可以操
作队列事件列表，可能会解除任务阻塞、可能会进行上下文切换，这也是不允许的。于是，解决办法是不但挂起调度器，还要给队列上锁，禁止任何中断来操作队列。

消息队列发送函数xQueueGenericSendFromISR()（中断）
''''''''''''''''''''''''''''''''''''''

既然有任务中发送消息的函数，当然也需要有在中断中发送消息函数，其实这个函数跟xQueueGenericSend()函数很像，只不过是执行的上下文环境是不一样的，xQueueGenericSendFromISR()函数只能用于中断中执行，是不带阻塞机制的，源码具体见代码清单15‑19。

代码清单15‑19xQueueGenericSendFromISR()函数源码

1 BaseType_t xQueueGenericSendFromISR( QueueHandle_t xQueue, **(1)**

2 const void \* const pvItemToQueue, **(2)**

3 BaseType_t \* const xHigherPriorityTaskWoken,\ **(3)**

4 const BaseType_t xCopyPosition ) **(4)**

5 {

6 BaseType_t xReturn;

7 UBaseType_t uxSavedInterruptStatus;

8 Queue_t \* const pxQueue = ( Queue_t \* ) xQueue;

9

10 /\* 已删除一些断言操作 \*/

11

12 uxSavedInterruptStatus = portSET_INTERRUPT_MASK_FROM_ISR();

13 {

14 /\* 队列未满 \*/

15 if ( ( pxQueue->uxMessagesWaiting < pxQueue->uxLength )

16 \|\| ( xCopyPosition == queueOVERWRITE ) ) { **(5)**

17 const int8_t cTxLock = pxQueue->cTxLock;

18 traceQUEUE_SEND_FROM_ISR( pxQueue );

19

20 /\* 完成消息拷贝 \*/

21 (void)prvCopyDataToQueue(pxQueue,pvItemToQueue,xCopyPosition );\ **(6)**

22

23 /\* 判断队列是否上锁 \*/

24 if ( cTxLock == queueUNLOCKED ) { **(7)**

25 /\* 已删除使用队列集部分代码 \*/

26 {

27 /\* 如果有任务在等待获取此消息队列 \*/

28 if ( listLIST_IS_EMPTY(

29 &( pxQueue->xTasksWaitingToReceive ) ) == pdFALSE ) {**(8)**

30 /\* 将任务从阻塞中恢复 \*/

31 if ( xTaskRemoveFromEventList(

32 &( pxQueue->xTasksWaitingToReceive )) != pdFALSE ) {**(9)**

33 if ( pxHigherPriorityTaskWoken != NULL ) {

34 /\* 解除阻塞的任务优先级比当前任务高,记录上下文切换请求,

35 等返回中断服务程序后,就进行上下文切换 \*/

36 \*pxHigherPriorityTaskWoken = pdTRUE; **(10)**

37 } else {

38 mtCOVERAGE_TEST_MARKER();

39 }

40 } else {

41 mtCOVERAGE_TEST_MARKER();

42 }

43 } else {

44 mtCOVERAGE_TEST_MARKER();

45 }

46 }

47

48 } else {

49 /\* 队列上锁,记录上锁次数,等到任务解除队列锁时,

50 使用这个计录数就可以知道有多少数据入队 \*/

51 pxQueue->cTxLock = ( int8_t ) ( cTxLock + 1 ); **(11)**

52 }

53

54 xReturn = pdPASS;

55 } else {

56 /\* 队列是满的，因为API执行的上下文环境是中断，

57 所以不能阻塞，直接返回队列已满错误代码errQUEUE_FULL \*/

58 traceQUEUE_SEND_FROM_ISR_FAILED( pxQueue ); **(12)**

59 xReturn = errQUEUE_FULL;

60 }

61 }

62 portCLEAR_INTERRUPT_MASK_FROM_ISR( uxSavedInterruptStatus );

63

64 return xReturn;

65 }

代码清单15‑19\ **(1)**\ ：消息队列句柄。

代码清单15‑19\ **(2)**\ ：指针，指向要发送的消息。

代码清单15‑19\ **(3)**\
：如果入队导致一个任务解锁，并且解锁的任务优先级高于当前运行的任务，则该函数将*pxHigherPriorityTaskWoken设置成pdTRUE。如果xQueueSendFromISR()设置这个值为pdTRUE，则中断退出前需要一次上下文切换。从FreeRTOS
V7.3.0起，pxHigherPriorityTaskWoken称为一个可选参数，并可以设置为NULL。

代码清单15‑19\ **(4)**\ ：发送数据到消息队列的位置，有以下3个选择，在queue.h中有定义，queueSEND_TO_BACK：发送到队尾；queueSEND_TO_FRONT：发送到队头；queueOVERWRITE：以覆盖的方式发送。

代码清单15‑19\ **(5)**\ ：判断队列是否已满，而如果是使用覆盖的方式发送数据，无论队列满或者没满，都可以发送。

代码清单15‑19\ **(6)**\ ：如果队列没满，可以调用prvCopyDataToQueue()函数将消息拷贝到消息队列中。

代码清单15‑19\ **(7)**\ ：判断队列是否上锁，如果队列上锁了，那么队列的等待接收列表就不能被访问。

代码清单15‑19\ **(8)**\ ：消息拷贝完毕，那么就看看有没有任务在等待消息，如果有任务在等待获取此消息，就要将任务从阻塞中恢复，

代码清单15‑19\ **(9)**\ ：调用xTaskRemoveFromEventList()函数将等待的任务从队列的等待接收列表xTasksWaitingToReceive中删除，并且添加到就绪列表中。

代码清单15‑19\ **(10)**\ ：如果恢复的任务优先级比当前运行任务的优先级高，那么需要记录上下文切换请求，等发送完成后，就进行一次任务切换。

代码清单15‑19\ **(11)**\ ：如果队列上锁，就记录上锁次数，等到任务解除队列锁时，从这个记录次数就可以知道有多少数据入队。

代码清单15‑19\ **(12)**\ ：队列是满的，因为API执行的上下文环境是中断，所以不能阻塞，直接返回队列已满错误代码errQUEUE_FULL。

xQueueGenericSendFromISR()函数没有阻塞机制，只能用于中断中发送消息，代码简单了很多，当成功入队后，如果有因为等待出队而阻塞的任务，系统会将该任务解除阻塞，要注意的是，解除了任务并不是会马上运行的，只是任务会被挂到就绪列表中。在执行解除阻塞操作之前，会判断队列是否上锁。如果没
有上锁，则可以解除被阻塞的任务，然后根据任务优先级情况来决定是否需要进行任务切换；如果队列已经上锁，则不能解除被阻塞的任务，只能是记录xTxLock的值，表示队列上锁期间消息入队的个数，也用来记录可以解除阻塞任务的个数，在队列解锁中会将任务解除阻塞。

从消息队列读取消息函数
^^^^^^^^^^^

当任务试图读队列中的消息时，可以指定一个阻塞超时时间，当且仅当消息队列中有消息的时候，任务才能读取到消息。在这段时间中，如果队列为空，该任务将保持阻塞状态以等待队列数据有效。当其他任务或中断服务程序往其等待的队列中写入了数据，该任务将自动由阻塞态转为就绪态。当任务等待的时间超过了指定的阻塞时间，即使
队列中尚无有效数据，任务也会自动从阻塞态转移为就绪态。

xQueueReceive()与xQueuePeek()
''''''''''''''''''''''''''''

代码清单15‑20xQueueReceive()函数原型

1 #define xQueueReceive( xQueue, pvBuffer, xTicksToWait ) \\

2 xQueueGenericReceive( ( xQueue ), ( pvBuffer ), \\

3 ( xTicksToWait ), pdFALSE )

xQueueReceive()是一个宏，宏展开是调用函数xQueueGenericReceive()。xQueueReceive()用于从一个队列中接收消息并把消息从队列中删除。接收的消息是以拷贝的形式进行的，所以我们必须提供一个足够大空间的缓冲区。具体能够拷贝多少数据到缓冲区，这个在队列创建的时候
已经设定。该函数绝不能在中断服务程序里面被调用，而是必须使用带有中断保护功能的xQueueReceiveFromISR ()来代替。xQueueReceive()函数的具体说明见表15‑7，应用实例见代码清单15‑21加粗部分。

表15‑7xQueueReceive()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xQu eueReceive(QueueHandle_t xQueue,  void \*pvBuffer,  TickType_t xTicksToWait);
     - |

   * - **功能**     |
     - 于从                   | 一个队列中接收消息，并把 | 接收的消息从队列中删除。 |
     - |

   * - **参数**     |
     - Queue                   |
     - 列句柄。               |

   * -
     - pvBuffer
     - 指针，                   | 指向接收到要保存的数据。 |

   * -
     - xTicksToWait
     - 队列                     | 空时，阻塞超时的最大时间 | 。如果该参数设置为0，函  | 数立刻返回。超时时间的单 | 位为系统节拍周期，常量po | rtTICK_PERIOD_MS用于辅助 | 计算真实的时间，单位为m  | s。如果INCLUDE_vTaskSusp |
       end设置成1，并且指定延时 | 为portMAX_DELAY将导致任  | 务无限阻塞（没有超时）。 |

   * - **返回值**   | 队
     - 项接收成功返回p      | dTRUE，否则返回pdFALSE。 |
     - |
             |


代码清单15‑21xQueueReceive()函数使用实例

1 static void Receive_Task(void\* parameter)

2 {

3 BaseType_t xReturn = pdTRUE;/\* 定义一个创建信息返回值，默认为pdPASS \*/

4 uint32_t r_queue; /\* 定义一个接收消息的变量 \*/

5 while (1) {

**6 xReturn = xQueueReceive( Test_Queue, /\* 消息队列的句柄 \*/**

**7 &r_queue, /\* 发送的消息内容 \*/**

**8 portMAX_DELAY); /\* 等待时间一直等 \*/**

**9 if (pdTRUE== xReturn)**

**10 printf("本次接收到的数据是：%d\n\n",r_queue);**

**11 else**

**12 printf("数据接收出错,错误代码: 0x%lx\n",xReturn);**

13 }

14 }

看到这里，有人就问了如果我接收了消息不想删除怎么办呢？其实，你能想到的东西，FreeRTOS看到也想到了，如果不想删除消息的话，就调用xQueuePeek()函数。

其实这个函数与xQueueReceive()函数的实现方式一样，连使用方法都一样，只不过xQueuePeek()函数接收消息完毕不会删除消息队列中的消息而已，函数原型具体见代码清单15‑22。

代码清单15‑22xQueuePeek()函数原型

1 #define xQueuePeek( xQueue, pvBuffer, xTicksToWait ) \\

2 xQueueGenericReceive( ( xQueue ), ( pvBuffer ), \\

3 ( xTicksToWait ), pdTRUE )

xQueueReceiveFromISR()与xQueuePeekFromISR()
''''''''''''''''''''''''''''''''''''''''''

xQueueReceiveFromISR()是xQueueReceive ()的中断版本，用于在中断服务程序中接收一个队列消息并把消息从队列中删除；xQueuePeekFromISR()是xQueuePeek()的中断版本，用于在中断中从一个队列中接收消息，但并不会把消息从队列中移除。

说白了这两个函数只能用于中断，是不带有阻塞机制的，并且是在中断中可以安全调用，函数说明具体见表15‑8与表15‑9，函数的使用实例具体见代码清单15‑23加粗部分。

表15‑8xQueueReceiveFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xQueueRece iveFromISR(QueueHandle_t xQueue,  void \*pvBuffer,  BaseType_t \*pxH igherPriorityTaskWoken);
     - |

   * - **功能**     |
     - 中                     | 断中从一个队列中接收消息 | ，并从队列中删除该消息。 |
     - |

   * - **参数**     |
     - Queue                   |
     - 列句柄。               |

   * -
     - pvBuffer
     - 指针，                   | 指向接收到要保存的数据。 |

   * -
     - p xHigherPriorityTaskWoken
     - 任务在往队               | 列投递信息时，如果队列满 | ，则任务将阻塞在该队列上 | 。如果xQueueReceiveFromI | SR()到账了一个任务解锁了 | 则将*pxHigherPriorityTas | kWoken设置为pdTRUE，否则 |
       *pxHigherPriorityTaskWok en的值将不变。从FreeRTOS | V7.3.0起，pxHigherPri    | orityTaskWoken作为一个可 | 选参数，可以设置为NULL。 |

   * - **返回值**   | 队
     - 项接收成功返回p      | dTRUE，否则返回pdFALSE。 |
     - |
             |


表15‑9xQueuePeekFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xQueueP eekFromISR(QueueHandle_t xQueue,  void \*pvBuffer);
     - |

   * - **功能**     |
     - 中断中从一             | 个队列中接收消息，但并不 | 会把消息从该队列中移除。 |
     - |

   * - **参数**     |
     - Queue                   |
     - 列句柄。               |

   * -
     - pvBuffer
     - 指针，                   | 指向接收到要保存的数据。 |

   * - **返回值**   | 队
     - | 列项接收(peek)成功返回p  | dTRUE，否则返回pdFALSE。 |
     - |


代码清单15‑23xQueueReceiveFromISR()函数使用实例

1 QueueHandle_t xQueue;

2

3 /\* 创建一个队列，并往队列里面发送一些数据 \*/

4 void vAFunction( void \*pvParameters )

5 {

6 char cValueToPost;

7 const TickType_t xTicksToWait = ( TickType_t )0xff;

8

9 /\* 创建一个可以容纳10个字符的队列 \*/

10 xQueue = xQueueCreate( 10, sizeof( char ) );

11 if ( xQueue == 0 ) {

12 /\* 队列创建失败 \*/

13 }

14

15 /\* ...
任务其他代码 \*/

16

17 /\* 往队列里面发送两个字符

18 如果队列满了则等待xTicksToWait个系统节拍周期*/

19 cValueToPost = 'a';

20 xQueueSend( xQueue, ( void \* ) &cValueToPost, xTicksToWait );

21 cValueToPost = 'b';

22 xQueueSend( xQueue, ( void \* ) &cValueToPost, xTicksToWait );

23

24 /\* 继续往队列里面发送字符

25 当队列满的时候该任务将被阻塞*/

26 cValueToPost = 'c';

27 xQueueSend( xQueue, ( void \* ) &cValueToPost, xTicksToWait );

28 }

29

30

31 /\* 中断服务程序：输出所有从队列中接收到的字符 \*/

32 void vISR_Routine( void )

33 {

34 BaseType_t xTaskWokenByReceive = pdFALSE;

35 char cRxedChar;

36

**37 while ( xQueueReceiveFromISR( xQueue,**

**38 ( void \* ) &cRxedChar,**

**39 &xTaskWokenByReceive) ) {**

40

41 /\* 接收到一个字符，然后输出这个字符 \*/

42 vOutputCharacter( cRxedChar );

43

44 /\* 如果从队列移除一个字符串后唤醒了向此队列投递字符的任务，

45 那么参数xTaskWokenByReceive将会设置成pdTRUE，这个循环无论重复多少次，

46 仅会有一个任务被唤醒 \*/

47 }

48

**49 if ( xTaskWokenByReceive != pdFALSE ) {**

**50 /\* 我们应该进行一次上下文切换，当ISR返回的时候则执行另外一个任务 \*/**

**51 /\* 这是一个上下文切换的宏，不同的处理器，具体处理的方式不一样 \*/**

**52 taskYIELD ();**

**53 }**

54}

从队列读取消息函数xQueueGenericReceive()
'''''''''''''''''''''''''''''''

由于在中断中接收消息的函数用的并不多，我们只讲解在任务中读取消息的函数——xQueueGenericReceive()，具体见代码清单15‑24。

代码清单15‑24xQueueGenericReceive()函数源码

1 /*-----------------------------------------------------------*/

2 BaseType_t xQueueGenericReceive( QueueHandle_t xQueue, **(1)**

3 void \* const pvBuffer, **(2)**

4 TickType_t xTicksToWait, **(3)**

5 const BaseType_t xJustPeeking ) **(4)**

6 {

7 BaseType_t xEntryTimeSet = pdFALSE;

8 TimeOut_t xTimeOut;

9 int8_t \*pcOriginalReadPosition;

10 Queue_t \* const pxQueue = ( Queue_t \* ) xQueue;

11

12 /\* 已删除一些断言 \*/

13 for ( ;; ) {

14 taskENTER_CRITICAL(); **(5)**

15 {

16 const UBaseType_t uxMessagesWaiting = pxQueue->uxMessagesWaiting;

17

18 /\* 看看队列中有没有消息 \*/

19 if ( uxMessagesWaiting > ( UBaseType_t ) 0 ) { **(6)**

20 /*防止仅仅是读取消息，而不进行消息出队操作*/

21 pcOriginalReadPosition = pxQueue->u.pcReadFrom; **(7)**

22 /\* 拷贝消息到用户指定存放区域pvBuffer \*/

23 prvCopyDataFromQueue( pxQueue, pvBuffer ); **(8)**

24

25 if ( xJustPeeking == pdFALSE ) { **(9)**

26 /\* 读取消息并且消息出队 \*/

27 traceQUEUE_RECEIVE( pxQueue );

28

29 /\* 获取了消息，当前消息队列的消息个数需要减一 \*/

30 pxQueue->uxMessagesWaiting = uxMessagesWaiting - 1;\ **(10)**

31 /\* 判断一下消息队列中是否有等待发送消息的任务 \*/

32 if ( listLIST_IS_EMPTY( **(11)**

33 &( pxQueue->xTasksWaitingToSend ) ) == pdFALSE ) {

34 /\* 将任务从阻塞中恢复 \*/

35 if ( xTaskRemoveFromEventList( **(12)**

36 &( pxQueue->xTasksWaitingToSend ) ) != pdFALSE ) {

37 /\* 如果被恢复的任务优先级比当前任务高，会进行一次任务切换 \*/

38 queueYIELD_IF_USING_PREEMPTION(); **(13)**

39 } else {

40 mtCOVERAGE_TEST_MARKER();

41 }

42 } else {

43 mtCOVERAGE_TEST_MARKER();

44 }

45 } else { **(14)**

46 /\* 任务只是看一下消息（peek），并不出队 \*/

47 traceQUEUE_PEEK( pxQueue );

48

49 /\* 因为是只读消息所以还要还原读消息位置指针 \*/

50 pxQueue->u.pcReadFrom = pcOriginalReadPosition;\ **(15)**

51

52 /\* 判断一下消息队列中是否还有等待获取消息的任务 \*/

53 if ( listLIST_IS_EMPTY( **(16)**

54 &( pxQueue->xTasksWaitingToReceive ) ) == pdFALSE ) {

55 /\* 将任务从阻塞中恢复 \*/

56 if ( xTaskRemoveFromEventList(

57 &( pxQueue->xTasksWaitingToReceive ) ) != pdFALSE ) {

58 /\* 如果被恢复的任务优先级比当前任务高，会进行一次任务切换 \*/

59 queueYIELD_IF_USING_PREEMPTION();

60 } else {

61 mtCOVERAGE_TEST_MARKER();

62 }

63 } else {

64 mtCOVERAGE_TEST_MARKER();

65 }

66 }

67

68 taskEXIT_CRITICAL(); **(17)**

69 return pdPASS;

70 } else { **(18)**

71 /\* 消息队列中没有消息可读 \*/

72 if ( xTicksToWait == ( TickType_t ) 0 ) { **(19)**

73 /\* 不等待，直接返回 \*/

74 taskEXIT_CRITICAL();

75 traceQUEUE_RECEIVE_FAILED( pxQueue );

76 return errQUEUE_EMPTY;

77 } else if ( xEntryTimeSet == pdFALSE ) {

78 /\* 初始化阻塞超时结构体变量，初始化进入

79 阻塞的时间xTickCount和溢出次数xNumOfOverflows \*/

80 vTaskSetTimeOutState( &xTimeOut ); **(20)**

81 xEntryTimeSet = pdTRUE;

82 } else {

83 mtCOVERAGE_TEST_MARKER();

84 }

85 }

86 }

87 taskEXIT_CRITICAL();

88

89 vTaskSuspendAll();

90 prvLockQueue( pxQueue ); **(21)**

91

92 /\* 检查超时时间是否已经过去了*/

93 if ( xTaskCheckForTimeOut( &xTimeOut, &xTicksToWait ) == pdFALSE ) {**(22)**

94 /\* 如果队列还是空的 \*/

95 if ( prvIsQueueEmpty( pxQueue ) != pdFALSE ) {

96 traceBLOCKING_ON_QUEUE_RECEIVE( pxQueue ); **(23)**

97 /\* 将当前任务添加到队列的等待接收列表中

98 以及阻塞延时列表，阻塞时间为用户指定的超时时间xTicksToWait \*/

99 vTaskPlaceOnEventList(

100 &( pxQueue->xTasksWaitingToReceive ), xTicksToWait );

101 prvUnlockQueue( pxQueue );

102 if ( xTaskResumeAll() == pdFALSE ) {

103 /\* 如果有任务优先级比当前任务高，会进行一次任务切换 \*/

104 portYIELD_WITHIN_API();

105 } else {

106 mtCOVERAGE_TEST_MARKER();

107 }

108 } else {

109 /\* 如果队列有消息了，就再试一次获取消息 \*/

110 prvUnlockQueue( pxQueue ); **(24)**

111 ( void ) xTaskResumeAll();

112 }

113 } else {

114 /\* 超时时间已过，退出 \*/

115 prvUnlockQueue( pxQueue ); **(25)**

116 ( void ) xTaskResumeAll();

117

118 if ( prvIsQueueEmpty( pxQueue ) != pdFALSE ) {

119 /\* 如果队列还是空的，返回错误代码errQUEUE_EMPTY \*/

120 traceQUEUE_RECEIVE_FAILED( pxQueue );

121 return errQUEUE_EMPTY; **(26)**

122 } else {

123 mtCOVERAGE_TEST_MARKER();

124 }

125 }

126 }

127 }

128 /*-----------------------------------------------------------*/

代码清单15‑24\ **(1)**\ ：消息队列句柄。

代码清单15‑24\ **(2)**\ ：指针，指向接收到要保存的数据。

代码清单15‑24\ **(3)**\ ：队列空时，用户指定的阻塞超时时间。如果该参数设置为0，函数立刻返回。超时时间的单位为系统节拍周期，常量portTICK_PERIOD_MS用于辅助计算真实的时间，单位为ms。如果INCLUDE_vTaskSuspend设置成1，并且指定延时为portMAX_
DELAY将导致任务无限阻塞（没有超时）。

代码清单15‑24\ **(4)**\ ：xJustPeeking用于标记消息是否需要出队，如果是pdFALSE，表示读取消息之后会进行出队操作，即读取消息后会把消息从队列中删除；如果是pdTRUE，则读取消息之后不会进行出队操作，消息还会保留在队列中。

代码清单15‑24\ **(5)**\ ：进入临界段。

代码清单15‑24\ **(6)**\ ：看看队列中有没有可读的消息。

代码清单15‑24\ **(7)**\ ：如果有消息，先记录读消息位置，防止仅仅是读取消息，而不进行消息出队操作

代码清单15‑24\ **(8)**\ ：拷贝消息到用户指定存放区域pvBuffer，pvBuffer由用户设置的，其空间大小必须不小于消息的大小。

代码清单15‑24\ **(9)**\ ：判断一下xJustPeeking的值，如果是pdFALSE，表示读取消息之后会进行出队操作。

代码清单15‑24\ **(10)**\ ：因为上面拷贝了消息到用户指定的数据区域，当前消息队列的消息个数需要减一。

代码清单15‑24\ **(11)**\ ：判断一下消息队列中是否有等待发送消息的任务。

代码清单15‑24\ **(12)**\ ：如果有任务在等待发送消息到这个队列，就要将任务从阻塞中恢复，调用xTaskRemoveFromEventList()函数将等待的任务从队列的等待发送列表xTasksWaitingToSend中删除，并且添加到就绪列表中。

代码清单15‑24\ **(13)**\ ：将任务从阻塞中恢复，如果恢复的任务优先级比当前运行任务的优先级高，那么需要进行一次任务切换。

代码清单15‑24\ **(14)**\ ：任务只是读取消息（xJustPeeking为pdTRUE），并不出队。

代码清单15‑24\ **(15)**\ ：因为是只读消息，所以还要还原读消息位置指针。

代码清单15‑24\ **(16)**\ ：判断一下消息队列中是否还有等待获取消息的任务，将那些任务恢复过来，如果恢复的任务优先级比当前运行任务的优先级高，那么需要进行一次任务切换。

代码清单15‑24\ **(17)**\ ：退出临界段。

代码清单15‑24\ **(18)**\ ：如果当前队列中没有可读的消息，那么系统会根据用户指定的阻塞超时时间xTicksToWait进行阻塞任务。

代码清单15‑24\ **(19)**\ ：xTicksToWait为0，那么不等待，直接返回errQUEUE_EMPTY。

代码清单15‑24\ **(20)**\ ：而如果用户指定了超时时间，系统就会初始化阻塞超时结构体变量，初始化进入阻塞的时间xTickCount和溢出次数xNumOfOverflows，为后面的阻塞任务做准备。

代码清单15‑24\ **(21)**\ ：因为前面进入了临界段，所以应先退出临界段，并且把调度器挂起，因为接下来的操作系统不允许其他任务访问队列，简单粗暴挂起调度器就不会进行任务切换，但是挂起调度器并不会禁止中断的发生，所以还需给队列上锁，因为系统不希望突然有中断操作这个队列的xTasksWait
ingToReceive列表和xTasksWaitingToSend列表。

代码清单15‑24\ **(22)**\ ：检查一下用户指定的超时时间是否已经过去了。如果没过则执行\ **(22)-(24)**\ 。

代码清单15‑24\ **(23)**\ ：如果队列还是空的，就将当前任务添加到队列的等待接收列表中以及阻塞延时列表，阻塞时间为用户指定的超时时间xTicksToWait，然后恢复调度器，如果调度器挂起期间有任务解除阻塞，并且解除阻塞的任务优先级比当前任务高，就需要进行一次任务切换。

代码清单15‑24\ **(24)**\ ：如果队列有消息了，就再试一次获取消息。

代码清单15‑24\ **(25)**\ ：超时时间已过，退出。

代码清单15‑24\ **(26)**\ ：返回错误代码errQUEUE_EMPTY。

消息队列使用注意事项
~~~~~~~~~~

在使用FreeRTOS提供的消息队列函数的时候，需要了解以下几点：

1. 使用xQueueSend()、xQueueSendFromISR()、xQueueReceive()等这些函数之前应先创建需消息队列，并根据队列句柄进行操作。

2. 队列读取采用的是先进先出（FIFO）模式，会先读取先存储在队列中的数据。当然也FreeRTOS也支持后进先出（LIFO）模式，那么读取的时候就会读取到后进队列的数据。

3. 在获取队列中的消息时候，我们必须要定义一个存储读取数据的地方，并且该数据区域大小不小于消息大小，否则，很可能引发地址非法的错误。

4. 无论是发送或者是接收消息都是以拷贝的方式进行，如果消息过于庞大，可以将消息的地址作为消息进行发送、接收。

5. 队列是具有自己独立权限的内核对象，并不属于任何任务。所有任务都可以向同一队列写入和读出。一个队列由多任务或中断写入是经常的事，但由多个任务读出倒是用的比较少。

消息队列实验
~~~~~~

消息队列实验是在FreeRTOS中创建了两个任务，一个是发送消息任务，一个是获取消息任务，两个任务独立运行，发送消息任务是通过检测按键的按下情况来发送消息，假如发送消息不成功，就把返回的错误情代码在串口打印出来，另一个任务是获取消息任务，在消息队列没有消息之前一直等待消息，一旦获取到消息就把消息打印
在串口调试助手里，具体见代码清单15‑25加粗部分。

代码清单15‑25消息队列实验

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS V9.0.0 + STM32 消息队列

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

27 /\* 开发板硬件bsp头文件 \*/

28 #include"bsp_led.h"

29 #include"bsp_usart.h"

30 #include"bsp_key.h"

31 /\* 任务句柄 \/

32 /\*

33 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

34 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

35 \* 这个句柄可以为NULL。

36 \*/

37 static TaskHandle_t AppTaskCreate_Handle = NULL;/\* 创建任务句柄 \*/

38 static TaskHandle_t Receive_Task_Handle = NULL;/\* LED任务句柄 \*/

39 static TaskHandle_t Send_Task_Handle = NULL;/\* KEY任务句柄 \*/

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

53 QueueHandle_t Test_Queue =NULL;

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

**65 #define QUEUE_LEN 4/\* 队列的长度，最大可包含多少个消息 \*/**

**66 #define QUEUE_SIZE 4/\* 队列中每个消息大小（字节） \*/**

67

68 /\*

69 \\*

70 \* 函数声明

71 \\*

72 \*/

73 static void AppTaskCreate(void);/\* 用于创建任务 \*/

74

75 static void Receive_Task(void\* pvParameters);/\* Receive_Task任务实现 \*/

76 static void Send_Task(void\* pvParameters);/\* Send_Task任务实现 \*/

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

94 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS消息队列实验！\n");

95 printf("按下KEY1或者KEY2发送队列消息\n");

96 printf("Receive任务接收到消息在串口回显\n\n");

97 /\* 创建AppTaskCreate任务 \*/

98 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate, /\* 任务入口函数 \*/

99 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

100 (uint16_t )512, /\* 任务栈大小 \*/

101 (void\* )NULL,/\* 任务入口函数参数 \*/

102 (UBaseType_t )1, /\* 任务的优先级 \*/

103 (TaskHandle_t\* )&AppTaskCreate_Handle);/\* 任务控制块指*/

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

**126 /\* 创建Test_Queue \*/**

**127 Test_Queue = xQueueCreate((UBaseType_t ) QUEUE_LEN,/\* 消息队列的长度 \*/**

**128 (UBaseType_t ) QUEUE_SIZE);/\* 消息的大小 \*/**

**129 if (NULL != Test_Queue)**

**130 printf("创建Test_Queue消息队列成功!\r\n");**

131

132 /\* 创建Receive_Task任务 \*/

133 xReturn = xTaskCreate((TaskFunction_t )Receive_Task,/\* 任务入口函数 \*/

134 (const char\* )"Receive_Task",/\* 任务名字 \*/

135 (uint16_t )512, /\* 任务栈大小 \*/

136 (void\* )NULL, /\* 任务入口函数参数 \*/

137 (UBaseType_t )2, /\* 任务的优先级 \*/

138 (TaskHandle_t\* )&Receive_Task_Handle);/*任务控制块指针*/

139 if (pdPASS == xReturn)

140 printf("创建Receive_Task任务成功!\r\n");

141

142 /\* 创建Send_Task任务 \*/

143 xReturn = xTaskCreate((TaskFunction_t )Send_Task, /\* 任务入口函数 \*/

144 (const char\* )"Send_Task",/\* 任务名字 \*/

145 (uint16_t )512, /\* 任务栈大小 \*/

146 (void\* )NULL,/\* 任务入口函数参数 \*/

147 (UBaseType_t )3, /\* 任务的优先级 \*/

148 (TaskHandle_t\* )&Send_Task_Handle);/*任务控制块指针 \*/

149 if (pdPASS == xReturn)

150 printf("创建Send_Task任务成功!\n\n");

151

152 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

153

154 taskEXIT_CRITICAL(); //退出临界区

155 }

156

157

158

159 /\*

160 \* @ 函数名： Receive_Task

161 \* @ 功能说明： Receive_Task任务主体

162 \* @ 参数：

163 \* @ 返回值：无

164 \/

**165 static void Receive_Task(void\* parameter)**

**166 {**

**167 BaseType_t xReturn = pdTRUE;/\* 定义一个创建信息返回值，默认为pdTRUE \*/**

**168 uint32_t r_queue; /\* 定义一个接收消息的变量 \*/**

**169 while (1) {**

**170 xReturn = xQueueReceive( Test_Queue, /\* 消息队列的句柄 \*/**

**171 &r_queue, /\* 发送的消息内容 \*/**

**172 portMAX_DELAY); /\* 等待时间一直等 \*/**

**173 if (pdTRUE == xReturn)**

**174 printf("本次接收到的数据是%d\n\n",r_queue);**

**175 else**

**176 printf("数据接收出错,错误代码: 0x%lx\n",xReturn);**

**177 }**

**178 }**

179

180 /\*

181 \* @ 函数名： Send_Task

182 \* @ 功能说明： Send_Task任务主体

183 \* @ 参数：

184 \* @ 返回值：无

185 \/

**186 static void Send_Task(void\* parameter)**

**187 {**

**188 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/**

**189 uint32_t send_data1 = 1;**

**190 uint32_t send_data2 = 2;**

**191 while (1) {**

**192 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**193 /\* KEY1 被按下 \*/**

**194 printf("发送消息send_data1！\n");**

**195 xReturn = xQueueSend( Test_Queue, /\* 消息队列的句柄 \*/**

**196 &send_data1,/\* 发送的消息内容 \*/**

**197 0 ); /\* 等待时间 0 \*/**

**198 if (pdPASS == xReturn)**

**199 printf("消息send_data1发送成功!\n\n");**

**200 }**

**201 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**202 /\* KEY2 被按下 \*/**

**203 printf("发送消息send_data2！\n");**

**204 xReturn = xQueueSend( Test_Queue, /\* 消息队列的句柄 \*/**

**205 &send_data2,/\* 发送的消息内容 \*/**

**206 0 ); /\* 等待时间 0 \*/**

**207 if (pdPASS == xReturn)**

**208 printf("消息send_data2发送成功!\n\n");**

**209 }**

**210 vTaskDelay(20);/\* 延时20个tick \*/**

**211 }**

**212 }**

213

214 /\*

215 \* @ 函数名： BSP_Init

216 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

217 \* @ 参数：

218 \* @ 返回值：无

219 \/

220 static void BSP_Init(void)

221 {

222 /\*

223 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

224 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

225 \* 都统一用这个优先级分组，千万不要再分组，切忌。

226 \*/

227 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

228

229 /\* LED 初始化 \*/

230 LED_GPIO_Config();

231

232 /\* 串口初始化 \*/

233 USART_Config();

234

235 /\* 按键初始化 \*/

236 Key_GPIO_Config();

237

238 }

239

240 /END OF FILE/

消息队列实验现象
~~~~~~~~

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，按下开发版的KEY1按键发
送消息1，按下KEY2按键发送消息2；我们按下KEY1试试，在串口调试助手中可以看到接收到消息1，我们按下KEY2试试，在串口调试助手中可以看到接收到消息2，具体见图15‑4。

|messag005|

图15‑4消息队列实验现象

.. |messag002| image:: media\messag002.png
   :width: 5.76806in
   :height: 2.18746in
.. |messag003| image:: media\messag003.png
   :width: 2.55195in
   :height: 1.97099in
.. |messag004| image:: media\messag004.png
   :width: 5.76806in
   :height: 6.1379in
.. |messag005| image:: media\messag005.png
   :width: 5.36688in
   :height: 3.01299in
