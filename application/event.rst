.. vim: syntax=rst

事件
========

事件的基本概念
~~~~~~~

事件是一种实现任务间通信的机制，主要用于实现多任务间的同步，但事件通信只能是事件类型的通信，无数据传输。与信号量不同的是，它可以实现一对多，多对多的同步。即一个任务可以等待多个事件的发生：可以是任意一个事件发生时唤醒任务进行事件处理；也可以是几个事件都发生后才唤醒任务进行事件处理。同样，也可以是多个
任务同步多个事件。

每一个事件组只需要很少的RAM空间来保存事件组的状态。事件组存储在一个EventBits_t类型的变量中，该变量在事件组结构体中定义。如果宏\ `configUSE_16_BIT_TICKS
<http://www.freertos.org/a00110.html#configUSE_16_BIT_TICKS>`__ 定义为1，那么变量uxEventBits就是16位的，其中有8个位用来存储事件组；而如果宏\ `configUSE_16_BIT_TICKS
<http://www.freertos.org/a00110.html#configUSE_16_BIT_TICKS>`__ 定义为0，那么变量uxEventBits就是32位的，其中有24个位用来存储事件组。在STM32中，我们一般将\ `configUSE_16_BIT_TICKS <http
://www.freertos.org/a00110.html#configUSE_16_BIT_TICKS>`__ 定义为0，那么uxEventBits是32位的，有24个位用来实现事件标志组。每一位代表一个事件，任务通过“逻辑与”或“逻辑或”与一个或多个事件建立关联，形成一个事件组。事件的“逻辑
或”也被称作是独立型同步，指的是任务感兴趣的所有事件任一件发生即可被唤醒；事件“逻辑与”则被称为是关联型同步，指的是任务感兴趣的若干事件都发生时才被唤醒，并且事件发生的时间可以不同步。

多任务环境下，任务、中断之间往往需要同步操作，一个事件发生会告知等待中的任务，即形成一个任务与任务、中断与任务间的同步。事件可以提供一对多、多对多的同步操作。一对多同步模型：一个任务等待多个事件的触发，这种情况是比较常见的；多对多同步模型：多个任务等待多个事件的触发。

任务可以通过设置事件位来实现事件的触发和等待操作。FreeRTOS的事件仅用于同步，不提供数据传输功能。

FreeRTOS提供的事件具有如下特点：

-  事件只与任务相关联，事件相互独立，一个32位的事件集合（EventBits_t类型的变量，实际可用与表示事件的只有24位），用于标识该任务发生的事件类型，其中每一位表示一种事件类型（0表示该事件类型未发生、1表示该事件类型已经发生），一共24种事件类型。

-  事件仅用于同步，不提供数据传输功能。

-  事件无排队性，即多次向任务设置同一事件(如果任务还未来得及读走)，等效于只设置一次。

-  允许多个任务对同一事件进行读写操作。

-  支持事件等待超时机制。

在FreeRTOS事件中，每个事件获取的时候，用户可以选择感兴趣的事件，并且选择读取事件信息标记，它有三个属性，分别是逻辑与，逻辑或以及是否清除标记。当任务等待事件同步时，可以通过任务感兴趣的事件位和事件信息标记来判断当前接收的事件是否满足要求，如果满足则说明任务等待到对应的事件，系统将唤醒等待的任
务；否则，任务会根据用户指定的阻塞超时时间继续等待下去。

事件的应用场景
~~~~~~~

FreeRTOS的事件用于事件类型的通信，无数据传输，也就是说，我们可以用事件来做标志位，判断某些事件是否发生了，然后根据结果做处理，那很多人又会问了，为什么我不直接用变量做标志呢，岂不是更好更有效率？非也非也，若是在裸机编程中，用全局变量是最为有效的方法，这点我不否认，但是在操作系统中，使用全局变
量就要考虑以下问题了：

-  如何对全局变量进行保护呢，如何处理多任务同时对它进行访问？

-  如何让内核对事件进行有效管理呢？使用全局变量的话，就需要在任务中轮询查看事件是否发送，这简直就是在浪费CPU资源啊，还有等待超时机制，使用全局变量的话需要用户自己去实现。

所以，在操作系统中，还是使用操作系统给我们提供的通信机制就好了，简单方便还实用。

在某些场合，可能需要多个时间发生了才能进行下一步操作，比如一些危险机器的启动，需要检查各项指标，当指标不达标的时候，无法启动，但是检查各个指标的时候，不能一下子检测完毕啊，所以，需要事件来做统一的等待，当所有的事件都完成了，那么机器才允许启动，这只是事件的其中一个应用。

事件可使用于多种场合，它能够在一定程度上替代信号量，用于任务与任务间，中断与任务间的同步。一个任务或中断服务例程发送一个事件给事件对象，而后等待的任务被唤醒并对相应的事件进行处理。但是它与信号量不同的是，事件的发送操作是不可累计的，而信号量的释放动作是可累计的。事件另外一个特性是，接收任务可等待多种
事件，即多个事件对应一个任务或多个任务。同时按照任务等待的参数，可选择是“逻辑或”触发还是“逻辑与”触发。这个特性也是信号量等所不具备的，信号量只能识别单一同步动作，而不能同时等待多个事件的同步。

各个事件可分别发送或一起发送给事件对象，而任务可以等待多个事件，任务仅对感兴趣的事件进行关注。当有它们感兴趣的事件发生时并且符合感兴趣的条件，任务将被唤醒并进行后续的处理动作。

事件运作机制
~~~~~~

接收事件时，可以根据感兴趣的参事件类型接收事件的单个或者多个事件类型。事件接收成功后，必须使用xClearOnExit选项来清除已接收到的事件类型，否则不会清除已接收到的事件，这样就需要用户显式清除事件位。用户可以自定义通过传入参数xWaitForAllBits选择读取模式，是等待所有感兴趣的事件还
是等待感兴趣的任意一个事件。

设置事件时，对指定事件写入指定的事件类型，设置事件集合的对应事件位为1，可以一次同时写多个事件类型，设置事件成功可能会触发任务调度。

清除事件时，根据入参数事件句柄和待清除的事件类型，对事件对应位进行清0操作。事件不与任务相关联，事件相互独立，一个32位的变量（事件集合，实际用于表示事件的只有24位），用于标识该任务发生的事件类型，其中每一位表示一种事件类型（0表示该事件类型未发生、1表示该事件类型已经发生），一共24种事件类型具
体见图18‑1。

|event002|

图18‑1事件集合set（一个32位的变量）

事件唤醒机制，当任务因为等待某个或者多个事件发生而进入阻塞态，当事件发生的时候会被唤醒，其过程具体见图18‑2。

|event003|

图18‑2事件唤醒任务示意图

任务1对事件3或事件5感兴趣（逻辑或），当发生其中的某一个事件都会被唤醒，并且执行相应操作。而任务2对事件3与事件5感兴趣（逻辑与），当且仅当事件3与事件5都发生的时候，任务2才会被唤醒，如果只有一个其中一个事件发生，那么任务还是会继续等待事件发生。如果接在收事件函数中设置了清除事件位xClearO
nExit，那么当任务唤醒后将把事件3和事件5的事件标志清零，否则事件标志将依然存在。

事件控制块
~~~~~

事件标志组存储在一个EventBits_t类型的变量中，该变量在事件组结构体中定义，具体见代码清单18‑1加粗部分。如果宏\ `configUSE_16_BIT_TICKS
<http://www.freertos.org/a00110.html#configUSE_16_BIT_TICKS>`__ 定义为1，那么变量uxEventBits就是16位的，其中有8个位用来存储事件组，如果宏\ `configUSE_16_BIT_TICKS <http://www.free
rtos.org/a00110.html#configUSE_16_BIT_TICKS>`__ 定义为0，那么变量uxEventBits就是32位的，其中有24个位用来存储事件组，每一位代表一个事件的发生与否，利用逻辑或、逻辑与等实现不同事件的不同唤醒处理。在STM32中，uxEventBits是3
2位的，所以我们有24个位用来实现事件组。除了事件标志组变量之外，FreeRTOS还使用了一个链表来记录等待事件的任务，所有在等待此事件的任务均会被挂载在等待事件列表xTasksWaitingForBits。

代码清单18‑1事件控制块

1 typedefstruct xEventGroupDefinition {

2 **EventBits_t uxEventBits;**

3 List_t xTasksWaitingForBits;

4

5 #if( configUSE_TRACE_FACILITY == 1 )

6 UBaseType_t uxEventGroupNumber;

7 #endif

8

9 #if( ( configSUPPORT_STATIC_ALLOCATION == 1 ) \\

10 && ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) )

11 uint8_t ucStaticallyAllocated;

12 #endif

13 } EventGroup_t;

事件函数接口讲解
~~~~~~~~

事件创建函数xEventGroupCreate()
^^^^^^^^^^^^^^^^^^^^^^^^^

xEventGroupCreate()用于创建一个事件组，并返回对应的句柄。要想使用该函数必须在头文件FreeRTOSConfig.h定义宏\ `configSUPPORT_DYNAMIC_ALLOCATION
<http://www.freertos.org/a00110.html#configSUPPORT_DYNAMIC_ALLOCATION>`__ 为1（在FreeRTOS.h中默认定义为1）且需要把FreeRTOS/source/event_groups.c 这个C文件添加到工程中。

每一个事件组只需要很少的RAM空间来保存事件的发生状态。如果使用函数xEventGroupCreate()来创建一个事件，那么需要的RAM是动态分配的。如果使用函数\ `xEventGroupCreateStatic
<http://www.freertos.org/xEventGroupCreateStatic.html>`__\ ()来创建一个事件，那么需要的RAM是静态分配的。我们暂时不讲解静态创建函数\ `xEventGroupCreateStatic
<http://www.freertos.org/xEventGroupCreateStatic.html>`__\ ()。

事件创建函数，顾名思义，就是创建一个事件，与其他内核对象一样，都是需要先创建才能使用的资源，FreeRTOS给我们提供了一个创建事件的函数xEventGroupCreate()，当创建一个事件时，系统会首先给我们分配事件控制块的内存空间，然后对该事件控制块进行基本的初始化，创建成功返回事件句柄；创建
失败返回NULL。所以，在使用创建函数之前，我们需要先定义有个事件的句柄，事件创建的源码具体见代码清单18‑2。

代码清单18‑2xEventGroupCreate()源码

1 #if( configSUPPORT_DYNAMIC_ALLOCATION == 1 )

2

3 EventGroupHandle_t xEventGroupCreate( void )

4 {

5 EventGroup_t \*pxEventBits;

6

7 /\* 分配事件控制块的内存 \*/

8 pxEventBits = ( EventGroup_t \* ) pvPortMalloc( sizeof( EventGroup_t ) );\ **(1)**

9

10 if ( pxEventBits != NULL ) { **(2)**

11 pxEventBits->uxEventBits = 0;

12 vListInitialise( &( pxEventBits->xTasksWaitingForBits ) );

13

14 #if( configSUPPORT_STATIC_ALLOCATION == 1 )

15 {

16 /\*

17 静态分配内存的，此处暂时不用理会

18 \*/

19 pxEventBits->ucStaticallyAllocated = pdFALSE;

20 }

21 #endif

22

23 traceEVENT_GROUP_CREATE( pxEventBits );

24 } else {

25 traceEVENT_GROUP_CREATE_FAILED();

26 }

27

28 return ( EventGroupHandle_t ) pxEventBits;

29 }

30

31 #endif

代码清单18‑2\ **(1)**\ ：因为事件标志组是FreeRTOS的内部资源，也是需要RAM的，所以，在创建的时候，会向系统申请一块内存，大小是事件控制块大小sizeof( EventGroup_t )。

代码清单18‑2\ **(2)**\
：如果分配内存成功，那么久对事件控制块的成员变量进行初始化，事件标志组变量清零，因为现在是创建事件，还没有事件发生，所以事件集合中所有位都为0，然后调用vListInitialise()函数将事件控制块中的等待事件列表进行初始化，该列表用于记录等待在此事件上的任务。

事件创建函数的源码都那么简单，其使用更为简单，不过需要我们在使用前定义一个指向事件控制块的指针，也就是常说的事件句柄，当事件创建成功，我们就可以根据我们定义的事件句柄来调用FreeRTOS的其他事件函数进行操作，具体见代码清单18‑3加粗部分。

代码清单18‑3事件创建函数xEventGroupCreate()使用实例

1 static EventGroupHandle_t Event_Handle =NULL;

2

3 /\* 创建 Event_Handle \*/

**4 Event_Handle = xEventGroupCreate();**

5 if (NULL != Event_Handle)

6 printf("Event_Handle 事件创建成功!\r\n");

7 else

8 /\* 创建失败，应为内存空间不足 \*/

事件删除函数vEventGroupDelete()
^^^^^^^^^^^^^^^^^^^^^^^^^

在很多场合，某些事件只用一次的，就好比在事件应用场景说的危险机器的启动，假如各项指标都达到了，并且机器启动成功了，那这个事件之后可能就没用了，那就可以进行销毁了。想要删除事件怎么办？FreeRTOS给我们提供了一个删除事件的函数——vEventGroupDelete()，使用它就能将事件进行删除了。
当系统不再使用事件对象时，可以通过删除事件对象控制块来释放系统资源，具体见代码清单18‑4。

代码清单18‑4vEventGroupDelete()源码

1 /*-----------------------------------------------------------*/

2 void vEventGroupDelete( EventGroupHandle_t xEventGroup )

3 {

4 EventGroup_t \*pxEventBits = ( EventGroup_t \* ) xEventGroup;

5 const List_t \*pxTasksWaitingForBits = &( pxEventBits->xTasksWaitingForBits );

6

7 vTaskSuspendAll(); **(1)**

8 {

9 traceEVENT_GROUP_DELETE( xEventGroup );

10 while(listCURRENT_LIST_LENGTH( pxTasksWaitingForBits )>(UBaseType_t )0)\ **(2)**

11 {

12 /\* 如果有任务阻塞在这个事件上，那么就要把事件从等待事件列表中移除 \*/

13 configASSERT( pxTasksWaitingForBits->xListEnd.pxNext

14 != ( ListItem_t \* ) &( pxTasksWaitingForBits->xListEnd ) );

15

16 ( void ) xTaskRemoveFromUnorderedEventList(

17 pxTasksWaitingForBits->xListEnd.pxNext,

18 eventUNBLOCKED_DUE_TO_BIT_SET ); **(3)**

19 }

20 #if( ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) \\

21 && ( configSUPPORT_STATIC_ALLOCATION == 0 ) )

22 {

23 /\* 释放事件的内存*/

24 vPortFree( pxEventBits ); **(4)**

25 }

26

27 /\* 已删除静态创建释放内存部分代码 \*/

28

29 #endif

30 }

31 ( void ) xTaskResumeAll(); **(5)**

32 }

33 /*-----------------------------------------------------------*/

代码清单18‑4\ **(1)**\ ：挂起调度器，因为接下来的操作不知道需要多长的时间，并且在删除的时候，不希望其他任务来操作这个事件标志组，所以暂时把调度器挂起，让当前任务占有CPU。

代码清单18‑4\ **(2)**\ ：当有任务被阻塞在事件等待列表中的时候，我们就要把任务恢复过来，否则删除了事件的话，就无法对事件进行读写操作，那这些任务可能永远等不到事件（因为任务有可能是一直在等待事件发生的），使用while循环保证所有的任务都会被恢复。

代码清单18‑4\ **(3)**\ ：调用xTaskRemoveFromUnorderedEventList()函数将任务从等待事件列表中移除，然后添加到就绪列表中，参与任务调度，当然，因为挂起了调度器，所以在这段时间里，即使是优先级更高的任务被添加到就绪列表，系统也不会进行任务调度，所以也就不会
影响当前任务删除事件的操作，这也是为什么需要挂起调度器的原因。但是，使用事件删除函数vEventGroupDelete()的时候需要注意，尽量在没有任务阻塞在这个事件的时候进行删除，否则任务无法等到正确的事件，因为删除之后，所有被恢复的任务都只能获得事件的值为0。

代码清单18‑4\ **(4)**\ ：释放事件的内存，因为在创建事件的时候申请了内存的，在不使用事件的时候就把内核还给系统。

代码清单18‑4\ **(5)**\ ：恢复调度器，之前的操作是恢复了任务，现在恢复调度器，那么处于就绪态的最高优先级任务将被运行。

vEventGroupDelete()用于删除由函数\ `xEventGroupCreate() <http://www.freertos.org/xEventGroupCreate.html>`__\
创建的事件组，只有被创建成功的事件才能被删除，但是需要注意的是该函数不允许在中断里面使用。当事件组被删除之后，阻塞在该事件组上的任务都会被解锁，并向等待事件的任务返回事件组的值为0，其使用是非常简单的，具体见代码清单18‑5加粗部分。

代码清单18‑5vEventGroupDelete函数使用实例

1 static EventGroupHandle_t Event_Handle =NULL;

2

3 /\* 创建 Event_Handle \*/

4 Event_Handle = xEventGroupCreate();

5 if (NULL != Event_Handle)

6 {

7 printf("Event_Handle 事件创建成功!\r\n");

8

**9 /\* 创建成功，可以删除 \*/**

**10 xEventGroupCreate(Event_Handle);**

11 } else

12 /\* 创建失败，应为内存空间不足 \*/

事件组置位函数xEventGroupSetBits()（任务）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

xEventGroupSetBits()用于置位事件组中指定的位，当位被置位之后，阻塞在该位上的任务将会被解锁。使用该函数接口时，通过参数指定的事件标志来设定事件的标志位，然后遍历等待在事件对象上的事件等待列表，判断是否有任务的事件激活要求与当前事件对象标志值匹配，如果有，则唤醒该任务。简单来说，就
是设置我们自己定义的事件标志位为1，并且看看有没有任务在等待这个事件，有的话就唤醒它。

注意的是该函数不允许在中断中使用，xEventGroupSetBits()的具体说明见表18‑1，源码具体见代码清单18‑6。

表18‑1xEventGroupSetBits()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ev
     - ntBits_t              | xEventGroupSetBits( EventGroupHandle_t xEventGroup,  const EventBits_t uxBitsToSet );
     - |

   * - **功能**     |
     - 位事件组中指定的位。   |
     - |

   * - **参数**     |
     - EventGroup              |
     - 件句柄。               |

   * -
     - uxBitsToSet
     - 指定事件中的事           | 件标志位。如设置uxBitsTo | Set为0x08则只置位位3，如 | 果设置uxBitsToSet为0x09  | 则位3和位0都需要被置位。 |

   * - **返回值**   | 返
     - 调用xEventGroupSe    | tBits() 时事件组中的值。 |
     - |


代码清单18‑6 xEventGroupSetBits()源码

1 /*-----------------------------------------------------------*/

2 EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup,

3 const EventBits_t uxBitsToSet )

4 {

5 ListItem_t \*pxListItem, \*pxNext;

6 ListItem_t const \*pxListEnd;

7 List_t \*pxList;

8 EventBits_t uxBitsToClear = 0, uxBitsWaitedFor, uxControlBits;

9 EventGroup_t \*pxEventBits = ( EventGroup_t \* ) xEventGroup;

10 BaseType_t xMatchFound = pdFALSE;

11

12 /\* 断言，判断事件是否有效 \*/

13 configASSERT( xEventGroup );

14 /\* 断言，判断要设置的事件标志位是否有效 \*/

15 configASSERT((uxBitsToSet&eventEVENT_BITS_CONTROL_BYTES ) == 0 );\ **(1)**

16

17 pxList = &( pxEventBits->xTasksWaitingForBits );

18 pxListEnd = listGET_END_MARKER( pxList );

19

20 vTaskSuspendAll(); **(2)**

21 {

22 traceEVENT_GROUP_SET_BITS( xEventGroup, uxBitsToSet );

23

24 pxListItem = listGET_HEAD_ENTRY( pxList );

25

26 /\* 设置事件标志位.
\*/

27 pxEventBits->uxEventBits \|= uxBitsToSet; **(3)**

28

29 /\* 设置这个事件标志位可能是某个任务在等待的事件

30 就遍历等待事件列表中的任务 \*/

31 while ( pxListItem != pxListEnd ) { **(4)**

32 pxNext = listGET_NEXT( pxListItem );

33 uxBitsWaitedFor = listGET_LIST_ITEM_VALUE( pxListItem );

34 xMatchFound = pdFALSE;

35

36 /\* 获取要等待事件的标记信息，是逻辑与还是逻辑或 \*/

37 uxControlBits = uxBitsWaitedFor & eventEVENT_BITS_CONTROL_BYTES;\ **(5)**

38 uxBitsWaitedFor &= ~eventEVENT_BITS_CONTROL_BYTES; **(6)**

39

40 /\* 如果只需要有一个事件标志位满足即可 \*/

41 if ((uxControlBits & eventWAIT_FOR_ALL_BITS ) == ( EventBits_t )0) {**(7)**

42 /\* 判断要等待的事件是否发生了 \*/

43 if ( ( uxBitsWaitedFor & pxEventBits->uxEventBits )

44 != ( EventBits_t ) 0 ) {

45 xMatchFound = pdTRUE; **(8)**

46 } else {

47 mtCOVERAGE_TEST_MARKER();

48 }

49 }

50 /\* 否则就要所有事件都发生的时候才能解除阻塞 \*/

51 else if ( ( uxBitsWaitedFor & pxEventBits->uxEventBits )

52 == uxBitsWaitedFor ) { **(9)**

53 /\* 所有事件都发生了 \*/

54 xMatchFound = pdTRUE;

55 } else { **(10)**

56 /\* Need all bits to be set, but not all the bits were set.
\*/

57 }

58

59 if ( xMatchFound != pdFALSE ) { **(11)**

60 /\* 找到了，然后看下是否需要清除标志位

61 如果需要，就记录下需要清除的标志位，等遍历完队列之后统一处理 \*/

62 if ( ( uxControlBits & eventCLEAR_EVENTS_ON_EXIT_BIT )

63 != ( EventBits_t ) 0 ) {

64 uxBitsToClear \|= uxBitsWaitedFor; **(12)**

65 } else {

66 mtCOVERAGE_TEST_MARKER();

67 }

68

69 /\* 将满足事件条件的任务从等待列表中移除，并且添加到就绪列表中 \*/

70 ( void ) xTaskRemoveFromUnorderedEventList( pxListItem,

71 pxEventBits->uxEventBits \| eventUNBLOCKED_DUE_TO_BIT_SET );\ **(13)**

72 }

73

74 /\* 循环遍历事件等待列表，可能不止一个任务在等待这个事件 \*/

75 pxListItem = pxNext; **(14)**

76 }

77

78 /\* 遍历完毕，清除事件标志位 \*/

79 pxEventBits->uxEventBits &= ~uxBitsToClear; **(15)**

80 }

81 ( void ) xTaskResumeAll(); **(16)**

82

83 return pxEventBits->uxEventBits; **(17)**

84 }

85 /*-----------------------------------------------------------*/

代码清单18‑6\ **(1)**\ ：断言，判断要设置的事件标志位是否有效，因为一个32位的事件标志组变量只有24位是用于设置事件的，而16位的事件标志组变量只有8位用于设置事件，高8位不允许设置事件，有其他用途，具体见代码清单18‑7

代码清单18‑7事件标志组高8位的用途

1 #if configUSE_16_BIT_TICKS == 1

2 #define eventCLEAR_EVENTS_ON_EXIT_BIT 0x0100U

3 #define eventUNBLOCKED_DUE_TO_BIT_SET 0x0200U

4 #define eventWAIT_FOR_ALL_BITS 0x0400U

5 #define eventEVENT_BITS_CONTROL_BYTES 0xff00U

6 #else

7 #define eventCLEAR_EVENTS_ON_EXIT_BIT 0x01000000UL

8 #define eventUNBLOCKED_DUE_TO_BIT_SET 0x02000000UL

9 #define eventWAIT_FOR_ALL_BITS 0x04000000UL

10 #define eventEVENT_BITS_CONTROL_BYTES 0xff000000UL

11 #endif

代码清单18‑6\ **(2)**\ ：挂起调度器，因为接下来的操作不知道需要多长的时间，因为需要遍历等待事件列表，并且有可能不止一个任务在等待事件，所以在满足任务等待的事件时候，任务允许被恢复，但是不允许运行，只有遍历完成的时候，任务才能被系统调度，在遍历期间，系统也不希望其他任务来操作这个事件标
志组，所以暂时把调度器挂起，让当前任务占有CPU。

代码清单18‑6\ **(3)**\ ：根据用户指定的uxBitsToSet设置事件标志位。

代码清单18‑6\ **(4)**\ ：设置这个事件标志位可能是某个任务在等待的事件，就需要遍历等待事件列表中的任务，看看这个事件是否与任务等待的事件匹配。

代码清单18‑6\ **(5)**\ ：获取要等待事件的标记信息，是逻辑与还是逻辑或。

代码清单18‑6\ **(6)**\ ：再获取任务的等待事件是什么。

代码清单18‑6\ **(7)**\ ：如果只需要有任意一个事件标志位满足唤醒任务（也是我们常说的“逻辑或”），那么还需要看看是否有这个事件发生了。

代码清单18‑6\ **(8)**\ ：判断要等待的事件是否发生了，发生了就需要把任务恢复，在这里记录一下要恢复的任务。

代码清单18‑6\ **(9)**\ ：如果任务等待的事件都要发生的时候（也是我们常说的“逻辑与”），就需要就要所有判断事件标志组中的事件是否都发生，如果是的话任务才能从阻塞中恢复，同样也需要标记一下要恢复的任务。

代码清单18‑6\ **(10)**\ ：这里是FreeRTOS暂时不用的，暂时不用理会。

代码清单18‑6\ **(11)**\ ：找到能恢复的任务，然后看下是否需要清除标志位，如果需要，就记录下需要清除的标志位，等遍历完队列之后统一处理，注意了，在一找到的时候不能清除，因为后面有可能一样有任务等着这个事件，只能在遍历任务完成之后才能清除事件标志位。

代码清单18‑6\ **(12)**\ ：运用或运算，标记一下要清除的事件标志位是哪些。

代码清单18‑6\ **(13)**\ ：将满足事件条件的任务从等待列表中移除，并且添加到就绪列表中。

代码清单18‑6\ **(14)**\ ：循环遍历事件等待列表，可能不止一个任务在等待这个事件。

代码清单18‑6\ **(15)**\ ：遍历完毕，清除事件标志位。

代码清单18‑6\ **(16)**\ ：恢复调度器，之前的操作是恢复了任务，现在恢复调度器，那么处于就绪态的最高优先级任务将被运行。

代码清单18‑6\ **(17)**\ ：返回用户设置的事件标志位值。

xEventGroupSetBits()的运用很简单，举个例子，比如我们要记录一个事件的发生，这个事件在事件组的位置是bit0，当它还未发生的时候，那么事件组bit0的值也是0，当它发生的时候，我们往事件集合bit0中写入这个事件，也就是0x01，那这就表示事件已经发生了，为了便于理解，一般操作我们
都是用宏定义来实现#define EVENT (0x01 << x)，“<< x”表示写入事件集合的bit x ，在使用该函数之前必须先创建事件，具体见代码清单18‑8加粗部分。

代码清单18‑8xEventGroupSetBits()函数使用实例

**1 #define KEY1_EVENT (0x01 << 0)//设置事件掩码的位0**

**2 #define KEY2_EVENT (0x01 << 1)//设置事件掩码的位1**

3

4 static EventGroupHandle_t Event_Handle =NULL;

5

**6 /\* 创建 Event_Handle \*/**

**7 Event_Handle = xEventGroupCreate();**

8 if (NULL != Event_Handle)

9 printf("Event_Handle 事件创建成功!\r\n");

10

11 static void KEY_Task(void\* parameter)

12 {

13 /\* 任务都是一个无限循环，不能返回 \*/

14 while (1) {

15 //如果KEY1被按下

16 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {

17 printf ( "KEY1被按下\n" );

18 /\* 触发一个事件1 \*/

**19 xEventGroupSetBits(Event_Handle,KEY1_EVENT);**

20 }

21

22 //如果KEY2被按下

23 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {

24 printf ( "KEY2被按下\n" );

25 /\* 触发一个事件2 \*/

**26 xEventGroupSetBits(Event_Handle,KEY2_EVENT);**

27 }

28 vTaskDelay(20); //每20ms扫描一次

29 }

30 }

事件组置位函数xEventGroupSetBitsFromISR()（中断）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

xEventGroupSetBitsFromISR()是xEventGroupSetBits()的中断版本，用于置位事件组中指定的位。置位事件组中的标志位是一个不确定的操作，因为阻塞在事件组的标志位上的任务的个数是不确定的。FreeRTOS是不允许不确定的操作在中断和临界段中发生的，所以xEvent
GroupSetBitsFromISR()给FreeRTOS的守护任务发送一个消息，让置位事件组的操作在守护任务里面完成，守护任务是基于调度锁而非临界段的机制来实现的。

需要注意的地方：正如上文提到的那样，在中断中事件标志的置位是在守护任务（也叫软件定时器服务任务）中完成的。因此FreeRTOS的守护任务与其他任务一样，都是系统调度器根据其优先级进行任务调度的，但守护任务的优先级必须比任何任务的优先级都要高，保证在需要的时候能立即切换任务从而达到快速处理的目的，因为
这是在中断中让事件标志位置位，其优先级由FreeRTOSConfig.h中的宏\ `configTIMER_TASK_PRIORITY <http://www.freertos.org/Configuring-a-real-time-RTOS-application-to-use-software-
timers.html>`__ 来定义。

其实xEventGroupSetBitsFromISR()函数真正调用的也是xEventGroupSetBits()函数，只不过是在守护任务中进行调用的，所以它实际上执行的上下文环境依旧是在任务中。

要想使用该函数，必须把configUSE_TIMERS 和 INCLUDE_xTimerPendFunctionCall 这些宏在FreeRTOSConfig.h中都定义为1，并且把FreeRTOS/source/event_groups.c 这个C文件添加到工程中编译。

xEventGroupSetBitsFromISR()函数的具体说明见表18‑2，其使用实例见代码清单18‑9加粗部分。

表18‑2xEventGroupSetBitsFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xEventGroupSetBitsFr omISR(EventGroupHandle_t xEventGroup,  const EventBits_t uxBitsToSet,  BaseType_t \*pxH
       igherPriorityTaskWoken);
     - |

   * - **功能**     |
     - 位事件组中指定         | 的位，在中断函数中使用。 |
     - |

   * - **参数**     |
     - EventGroup              |
     - 件句柄。               |

   * -
     - uxBitsToSet
     - 指定事件组中的哪些位     | 需要置位。如设置uxBitsTo | Set为0x08则只置位位3，如 | 果设置uxBitsToSet为0x09  | 则位3和位0都需要被置位。 |

   * -
     - p xHigherPriorityTaskWoken
     - pxHigherPriorityTaskW oken在使用之前必须初始化 | 成pdFALSE。调用xEventGro | upSetBitsFromISR()会给守 | 护任务发送一个消息，如果 | 守护任务的优先级高于当前 | 被中断的任务的优先级的话 | （一般情况下都需要将守护 |
       任务的优先级设置为所有任 | 务中最高优先级），pxHig  | herPriorityTaskWoken会被 | 置为pdTRUE，然后在中断退 | 出前执行一次上下文切换。 |

   * - **返回值**   | 消
     - 成功发送给守护任     | 务之后则返回pdTRUE，否则 | 返回pdFAIL。如果定时器服 | 务队列满了将返回pdFAIL。 |
     - |
           |
           |
           |


代码清单18‑9xEventGroupSetBitsFromISR()函数使用实例

1 #define BIT_0 ( 1 << 0 )

2 #define BIT_4 ( 1 << 4 )

3

4 /\* 假定事件组已经被创建 \*/

5 EventGroupHandle_t xEventGroup;

6

7 /\* 中断ISR \*/

8 void anInterruptHandler( void )

9 {

10 BaseType_t xHigherPriorityTaskWoken, xResult;

11

**12 /\* xHigherPriorityTaskWoken在使用之前必须先初始化为pdFALSE \*/**

**13 xHigherPriorityTaskWoken = pdFALSE;**

**14**

**15 /\* 置位事件组xEventGroup的的Bit0 和Bit4 \*/**

**16 xResult = xEventGroupSetBitsFromISR(**

**17 xEventGroup,**

**18 BIT_0 \| BIT_4,**

**19 &xHigherPriorityTaskWoken );**

20

21 /\* 信息是否发送成功 \*/

22 if ( xResult != pdFAIL ) {

23 /\* 如果xHigherPriorityTaskWoken 的值为 pdTRUE

24 则进行一次上下文切换*/

**25 portYIELD_FROM_ISR( xHigherPriorityTaskWoken );**

26 }

27 }

等待事件函数xEventGroupWaitBits()
^^^^^^^^^^^^^^^^^^^^^^^^^^^

既然标记了事件的发生，那么我怎么知道他到底有没有发生，这也是需要一个函数来获取事件是否已经发生，FreeRTOS提供了一个等待指定事件的函数——xEventGroupWaitBits()，通过这个函数，任务可以知道事件标志组中的哪些位，有什么事件发生了，然后通过“逻辑与”、“逻辑或”等操作对感兴趣的
事件进行获取，并且这个函数实现了等待超时机制，当且仅当任务等待的事件发生时，任务才能获取到事件信息。在这段时间中，如果事件一直没发生，该任务将保持阻塞状态以等待事件发生。当其他任务或中断服务程序往其等待的事件设置对应的标志位，该任务将自动由阻塞态转为就绪态。当任务等待的时间超过了指定的阻塞时间，即使
事件还未发生，任务也会自动从阻塞态转移为就绪态。这样子很有效的体现了操作系统的实时性，如果事件正确获取（等待到）则返回对应的事件标志位，由用户判断再做处理，因为在事件超时的时候也会返回一个不能确定的事件值，所以需要判断任务所等待的事件是否真的发生。

EventGroupWaitBits()用于获取事件组中的一个或多个事件发生标志，当要读取的事件标志位没有被置位时任务将进入阻塞等待状态。要想使用该函数必须把FreeRTOS/source/event_groups.c 这个C文件添加到工程中。xEventGroupWaitBits()的具体说明见表
18‑3，源码具体见代码清单18‑10。

表18‑3xEventGroupWaitBits()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ev
     - ntBits_t              | x EventGroupWaitBits(const EventGroupHandle_t xEventGroup,  const EventBits_t uxBitsToWaitFor,  const BaseType_t xClearOnExit,
       const BaseType_t xWaitForAllBits,  TickType_t xTicksToWait );
     - |

   * - **功能**     |
     - | 于获取任务感兴趣的事件。 |
     - |

   * - **参数**     |
     - EventGroup              |
     - 件句柄。               |

   * -
     - uxBitsToWaitFor
     - 一个按位或的值，         | 指定需要等待事件组中的哪 | 些位置1。如果需要等待bit | 0 and/or bit 2那么ux                  | BitsToWaitFor配置为0x05( | 0101b)。如果需要等待bits | 0 and/or bit 1
       and/or bit 2那么uxBitsToWa          | itFor配置为0x07(0111b)。 |

   * -
     - xClearOnExit
     - pdTRUE：当xEv            | entGroupWaitBits()等待到 | 满足任务唤醒的事件时，系 | 统将清除由形参uxBitsToW  | aitFor指定的事件标志位。 |

       pdFALSE：                | 不会清除由形参uxBitsToW  | aitFor指定的事件标志位。 |

   * -
     - xWaitForAllBits
     - pdTRUE：当形参uxBitsToW  | aitFor指定的位都置位的时 | 候，xEventGroupWaitBits  | ()才满足任务唤醒的条件， | 这也是“逻辑与”等待事件， | 并且在没有超时的情况下返 | 回对应的事件标志位的值。 |

       pdFALSE：当形参uxBitsT   | oWaitFor指定的位有其中任 | 意一个置位的时候，这也是 | 常说的“逻辑或”等待事件， | 在没有超时的情况下函数返 | 回对应的事件标志位的值。 |

   * -
     - xTicksToWait
     - 最大超时                 | 时间，单位为系统节拍周期 | ，常量portTICK_PERIOD_MS | 用于辅助把时间转换成MS。 |

   * - **返回值**   | 返
     - | 回事件中的哪些事件标志位 | 被置位，返回值很可能并不 | 是用户指定的事件位，需要 | 对返回值进行判断再处理。 |
     - |


代码清单18‑10xEventGroupWaitBits()源码

1 /*-----------------------------------------------------------*/

2 EventBits_t xEventGroupWaitBits( EventGroupHandle_t xEventGroup,

3 const EventBits_t uxBitsToWaitFor,

4 const BaseType_t xClearOnExit,

5 const BaseType_t xWaitForAllBits,

6 TickType_t xTicksToWait )

7 {

8 EventGroup_t \*pxEventBits = ( EventGroup_t \* ) xEventGroup;

9 EventBits_t uxReturn, uxControlBits = 0;

10 BaseType_t xWaitConditionMet, xAlreadyYielded;

11 BaseType_t xTimeoutOccurred = pdFALSE;

12

13 /\* 断言 \*/

14 configASSERT( xEventGroup );

15 configASSERT( ( uxBitsToWaitFor & eventEVENT_BITS_CONTROL_BYTES ) == 0 );

16 configASSERT( uxBitsToWaitFor != 0 );

17 #if ( ( INCLUDE_xTaskGetSchedulerState == 1 ) \|\| ( configUSE_TIMERS == 1 ) )

18 {

19 configASSERT( !( ( xTaskGetSchedulerState()

20 == taskSCHEDULER_SUSPENDED ) && ( xTicksToWait != 0 ) ) );

21 }

22 #endif

23

24 vTaskSuspendAll(); **(1)**

25 {

26 const EventBits_t uxCurrentEventBits = pxEventBits->uxEventBits;

27

28 /\* 先看下当前事件中的标志位是否已经满足条件了 \*/

29 xWaitConditionMet = prvTestWaitCondition( uxCurrentEventBits,

30 uxBitsToWaitFor,

31 xWaitForAllBits ); **(2)**

32

33 if ( xWaitConditionMet != pdFALSE ) { **(3)**

34 /\* 满足条件了，就可以直接返回了，注意这里返回的是的当前事件的所有标志位 \*/

35 uxReturn = uxCurrentEventBits;

36 xTicksToWait = ( TickType_t ) 0;

37

38 /\* 看看在退出的时候是否需要清除对应的事件标志位 \*/

39 if ( xClearOnExit != pdFALSE ) { **(4)**

40 pxEventBits->uxEventBits &= ~uxBitsToWaitFor;

41 } else {

42 mtCOVERAGE_TEST_MARKER();

43 }

44 }

45 /\* 不满足条件，并且不等待 \*/

46 else if ( xTicksToWait == ( TickType_t ) 0 ) { **(5)**

47 /\* 同样也是返回当前事件的所有标志位 \*/

48 uxReturn = uxCurrentEventBits;

49 }

50 /\* 用户指定超时时间了，那就进入等待状态 \*/

51 else { **(6)**

52 /\* 保存一下当前任务的信息标记，以便在恢复任务的时候对事件进行相应的操作 \*/

53 if ( xClearOnExit != pdFALSE ) {

54 uxControlBits \|= eventCLEAR_EVENTS_ON_EXIT_BIT;

55 } else {

56 mtCOVERAGE_TEST_MARKER();

57 }

58

59 if ( xWaitForAllBits != pdFALSE ) {

60 uxControlBits \|= eventWAIT_FOR_ALL_BITS;

61 } else {

62 mtCOVERAGE_TEST_MARKER();

63 }

64

65 /\* 当前任务进入事件等待列表中，任务将被阻塞指定时间xTicksToWait \*/

66 vTaskPlaceOnUnorderedEventList(

67 &( pxEventBits->xTasksWaitingForBits ),

68 ( uxBitsToWaitFor \| uxControlBits ),

69 xTicksToWait ); **(7)**

70

71 uxReturn = 0;

72

73 traceEVENT_GROUP_WAIT_BITS_BLOCK( xEventGroup,

74 uxBitsToWaitFor );

75 }

76 }

77 xAlreadyYielded = xTaskResumeAll(); **(8)**

78

79 if ( xTicksToWait != ( TickType_t ) 0 ) {

80 if ( xAlreadyYielded == pdFALSE ) {

81 /\* 进行一次任务切换 \*/

82 portYIELD_WITHIN_API(); **(9)**

83 } else {

84 mtCOVERAGE_TEST_MARKER();

85 }

86

87 /\* 进入到这里说明当前的任务已经被重新调度了 \*/

88

89 uxReturn = uxTaskResetEventItemValue(); **(10)**

90

91 if ( ( uxReturn & eventUNBLOCKED_DUE_TO_BIT_SET )

92 == ( EventBits_t ) 0 ) { **(11)**

93 taskENTER_CRITICAL();

94 {

95 /\* 超时返回时，直接返回当前事件的所有标志位 \*/

96 uxReturn = pxEventBits->uxEventBits;

97

98 /\* 再判断一次是否发生了事件 \*/

99 if ( prvTestWaitCondition(uxReturn, **(12)**

100 uxBitsToWaitFor,

101 xWaitForAllBits )!= pdFALSE) {

102 /\* 如果发生了，那就清除事件标志位并且返回 \*/

103 if ( xClearOnExit != pdFALSE ) {

104 pxEventBits->uxEventBits &= ~uxBitsToWaitFor;\ **(13)**

105 } else {

106 mtCOVERAGE_TEST_MARKER();

107 }

108 } else {

109 mtCOVERAGE_TEST_MARKER();

110 }

111 }

112 taskEXIT_CRITICAL();

113

114 xTimeoutOccurred = pdFALSE;

115 } else {

116

117 }

118

119 /\* 返回事件所有标志位 \*/

120 uxReturn &= ~eventEVENT_BITS_CONTROL_BYTES; **(14)**

121 }

122 traceEVENT_GROUP_WAIT_BITS_END( xEventGroup,

123 uxBitsToWaitFor,

124 xTimeoutOccurred );

125

126 return uxReturn;

127 }

128 /*-----------------------------------------------------------*/

代码清单18‑10\ **(1)**\ ：挂起调度器。

代码清单18‑10\ **(2)**\ ：先看下当前事件中的标志位是否已经满足条件了任务等待的事件， prvTestWaitCondition()函数其实就是判断一下用户等待的事件是否与当前事件标志位一致。

代码清单18‑10\ **(3)**\ ：满足条件了，就可以直接返回了，注意这里返回的是的当前事件的所有标志位，所以这是一个不确定的值，需要用户自己判断一下是否满足要求。然后把用户指定的等待超时时间xTicksToWait也重置为0，这样子等下就能直接退出函数返回了。

代码清单18‑10\ **(4)**\ ：看看在退出的时候是否需要清除对应的事件标志位，如果xClearOnExit为pdTRUE则需要清除事件标志位，如果为pdFALSE就不需要清除。

代码清单18‑10\ **(5)**\ ：当前事件中不满足任务等待的事件，并且用户指定不进行等待，那么可以直接退出，同样也会返回当前事件的所有标志位，所以在使用xEventGroupWaitBits()函数的时候需要对返回值做判断，保证等待到的事件是任务需要的事件。

代码清单18‑10\ **(6)**\ ：而如果用户指定超时时间了，并且当前事件不满足任务的需求，那任务就进入等待状态以等待事件的发生。

代码清单18‑10\ **(7)**\ ：将当前任务添加到事件等待列表中，任务将被阻塞指定时间xTicksToWait，并且这个列表项的值是用于保存任务等待事件需求的信息标记，以便在事件标志位置位的时候对等待事件的任务进行相应的操作。

代码清单18‑10\ **(8)**\ ：恢复调度器。

代码清单18‑10\ **(9)**\ ：在恢复调度器的时候，如果有更高优先级的任务恢复了，那么就进行一次任务的切换。

代码清单18‑10\ **(10)**\ ：程序能进入到这里说明当前的任务已经被重新调度了，调用uxTaskResetEventItemValue()返回并重置xEventListItem的值，因为之前事件列表项的值被保存起来了，现在取出来看看是不是有事件发生。

代码清单18‑10\ **(11)**\ ：如果仅仅是超时返回，那系统就会直接返回当前事件的所有标志位。

代码清单18‑10\ **(12)**\ ：再判断一次是否发生了事件。

代码清单18‑10\ **(13)**\ ：如果发生了，那就清除事件标志位并且返回。

代码清单18‑10\ **(14)**\ ：否则就返回事件所有标志位，然后退出。

下面简单分析处理过程：当用户调用这个函数接口时，系统首先根据用户指定参数和接收选项来判断它要等待的事件是否发生，如果已经发生，则根据参数xClearOnExit来决定是否清除事件的相应标志位，并且返回事件标志位的值，但是这个值并不是一个稳定的值，所以在等待到对应事件的时候，还需我们判断事件是否与任务
需要的一致；如果事件没有发生，则把任务添加到事件等待列表中，把任务感兴趣的事件标志值和等待选项填用列表项的值来表示，直到事件发生或等待时间超时，事件等待函数xEventGroupWaitBits()具体用法见代码清单18‑11加粗部分。

代码清单18‑11xEventGroupWaitBits()使用实例

1 static void LED_Task(void\* parameter)

2 {

3 EventBits_t r_event; /\* 定义一个事件接收变量 \*/

4 /\* 任务都是一个无限循环，不能返回 \*/

5 while (1) {

6 /\*

7 \* 等待接收事件标志

8 \*

9 \* 如果xClearOnExit设置为pdTRUE，那么在xEventGroupWaitBits()返回之前，

10 \* 如果满足等待条件（如果函数返回的原因不是超时），那么在事件组中设置

11 \* 的uxBitsToWaitFor中的任何位都将被清除。

12 \* 如果xClearOnExit设置为pdFALSE，

13 \* 则在调用xEventGroupWaitBits()时，不会更改事件组中设置的位。

14 \*

15 \* xWaitForAllBits如果xWaitForAllBits设置为pdTRUE，则当uxBitsToWaitFor中

16 \* 的所有位都设置或指定的块时间到期时，xEventGroupWaitBits()才返回。

17 \* 如果xWaitForAllBits设置为pdFALSE，则当设置uxBitsToWaitFor中设置的任何

18 \* 一个位置1 或指定的块时间到期时，xEventGroupWaitBits()都会返回。

19 \* 阻塞时间由xTicksToWait参数指定。

20 \/

**21 r_event = xEventGroupWaitBits(Event_Handle, /\* 事件对象句柄 \*/**

**22 KEY1_EVENT|KEY2_EVENT,/\* 接收任务感兴趣的事件 \*/**

**23 pdTRUE, /\* 退出时清除事件位 \*/**

**24 pdTRUE, /\* 满足感兴趣的所有事件 \*/**

**25 portMAX_DELAY);/\* 指定超时事件,一直等 \*/**

**26**

**27 if ((r_event & (KEY1_EVENT|KEY2_EVENT)) == (KEY1_EVENT|KEY2_EVENT)) {**

**28 /\* 如果接收完成并且正确 \*/**

29 printf ( "KEY1与KEY2都按下\n");

30 LED1_TOGGLE; //LED1 反转

31 } else

32 printf ( "事件错误！\n");

33 }

34 }

xEventGroupClearBits()与xEventGroupClearBitsFromISR()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

xEventGroupClearBits()与xEventGroupClearBitsFromISR()都是用于清除事件组指定的位，如果在获取事件的时候没有将对应的标志位清除，那么就需要用这个函数来进行显式清除，xEventGroupClearBits()函数不能在中断中使用，而是由具有中断保护功能
的\ `xEventGroupClearBitsFromISR()
<http://www.freertos.org/xEventGroupSetBitsFromISR.html>`__ 来代替，中断清除事件标志位的操作在守护任务（也叫定时器服务任务）里面完成。守护进程的优先级由FreeRTOSConfig.h中的宏\
`configTIMER_TASK_PRIORITY <http://www.freertos.org/Configuring-a-real-time-RTOS-application-to-use-software-
timers.html>`__ 来定义。要想使用该函数必须把FreeRTOS/source/event_groups.c 这个C文件添加到工程中。xEventGroupClearBits()的具体说明见表18‑4，应用举例见代码清单18‑12加粗部分。

表18‑4xEventGroupClearBits()与xEventGroupClearBitsFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ev
     - ntBits_t              | xEventGroupClea rBits(EventGroupHandle_t xEventGroup,  const EventBits_t uxBitsToClear );  BaseType_t xEventGroupClearBitsFr
       omISR(EventGroupHandle_t xEventGroup,  const EventBits_t uxBitsToClear );
     - |

   * - **功能**     |
     - 除事件组中指定的位。   |
     - |

   * - **参数**     |
     - EventGroup              |
     - 件句柄。               |

   * -
     - uxBitsToClear
     - 指定事件组中的哪个位     | 需要清除。如设置uxBitsTo | Set为0x08则只清除位3，如 | 果设置uxBitsToSet为0x09  | 则位3和位0都需要被清除。 |

   * - **返回值**   | 事
     - 在还                 | 没有清除指定位之前的值。 |
     - |


注：由于这两个源码过于简单，就不讲解。

代码清单18‑12xEventGroupClearBits()函数使用实例

1 #define BIT_0 ( 1 << 0 )

2 #define BIT_4 ( 1 << 4 )

3

4 void aFunction( EventGroupHandle_t xEventGroup )

5 {

6 EventBits_t uxBits;

7

**8 /\* 清楚事件组的 bit 0 and bit 4 \*/**

**9 uxBits = xEventGroupClearBits(**

**10 xEventGroup,**

**11 BIT_0 \| BIT_4 );**

12

13 if ( ( uxBits & ( BIT_0 \| BIT_4 ) ) == ( BIT_0 \| BIT_4 ) ) {

14 /\* 在调用xEventGroupClearBits()之前bit0和bit4都置位

15 但是现在是被清除了*/

16 } else if ( ( uxBits & BIT_0 ) != 0 ) {

17 /\* 在调用xEventGroupClearBits()之前bit0已经置位

18 但是现在是被清除了*/

19 } else if ( ( uxBits & BIT_4 ) != 0 ) {

20 /\* 在调用xEventGroupClearBits()之前bit4已经置位

21 但是现在是被清除了*/

22 } else {

23 /\* 在调用xEventGroupClearBits()之前bit0和bit4都没被置位 \*/

24 }

25 }

事件实验
~~~~

事件标志组实验是在FreeRTOS中创建了两个任务，一个是设置事件任务，一个是等待事件任务，两个任务独立运行，设置事件任务通过检测按键的按下情况设置不同的事件标志位，等待事件任务则获取这两个事件标志位，并且判断两个事件是否都发生，如果是则输出相应信息，LED进行翻转。等待事件任务的等待时间是port
MAX_DELAY，一直在等待事件的发生，等待到事件之后清除对应的事件标记位，具体见代码清单18‑13加粗部分。

代码清单18‑13事件实验

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS V9.0.0 + STM32 事件

8 \\*

9 \* @attention

10 \*

11 \* 实验平台:野火STM32全系列开发板

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

31 /\* 任务句柄 \/

32 /\*

33 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

34 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

35 \* 这个句柄可以为NULL。

36 \*/

37 static TaskHandle_t AppTaskCreate_Handle = NULL;/\* 创建任务句柄 \*/

38 static TaskHandle_t LED_Task_Handle = NULL;/\* LED_Task任务句柄 \*/

39 static TaskHandle_t KEY_Task_Handle = NULL;/\* KEY_Task任务句柄 \*/

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

**53 static EventGroupHandle_t Event_Handle =NULL;**

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

**65 #define KEY1_EVENT (0x01 << 0)//设置事件掩码的位0**

**66 #define KEY2_EVENT (0x01 << 1)//设置事件掩码的位1**

67

68 /\*

69 \\*

70 \* 函数声明

71 \\*

72 \*/

73 static void AppTaskCreate(void);/\* 用于创建任务 \*/

74

75 static void LED_Task(void\* pvParameters);/\* LED_Task 任务实现 \*/

76 static void KEY_Task(void\* pvParameters);/\* KEY_Task 任务实现 \*/

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

94 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS事件标志位实验！\n");

95 /\* 创建AppTaskCreate任务 \*/

96 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,/\* 任务入口函数 \*/

97 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

98 (uint16_t )512, /\* 任务栈大小 \*/

99 (void\* )NULL,/\* 任务入口函数参数 \*/

100 (UBaseType_t )1, /\* 任务的优先级 \*/

101 (TaskHandle_t\* )&AppTaskCreate_Handle);/*任务控制块指针 \*/

102 /\* 启动任务调度 \*/

103 if (pdPASS == xReturn)

104 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

105 else

106 return -1;

107

108 while (1); /\* 正常不会执行到这里 \*/

109 }

110

111

112 /\*

113 \* @ 函数名： AppTaskCreate

114 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

115 \* @ 参数：无

116 \* @ 返回值：无

117 \/

118 static void AppTaskCreate(void)

119 {

120 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

121

122 taskENTER_CRITICAL(); //进入临界区

123

**124 /\* 创建 Event_Handle \*/**

**125 Event_Handle = xEventGroupCreate();**

**126 if (NULL != Event_Handle)**

**127 printf("Event_Handle 事件创建成功!\r\n");**

128

129 /\* 创建LED_Task任务 \*/

130 xReturn = xTaskCreate((TaskFunction_t )LED_Task, /\* 任务入口函数 \*/

131 (const char\* )"LED_Task",/\* 任务名字 \*/

132 (uint16_t )512, /\* 任务栈大小 \*/

133 (void\* )NULL, /\* 任务入口函数参数 \*/

134 (UBaseType_t )2, /\* 任务的优先级 \*/

135 (TaskHandle_t\* )&LED_Task_Handle);/\* 任务控制块指针 \*/

136 if (pdPASS == xReturn)

137 printf("创建LED_Task任务成功!\r\n");

138

139 /\* 创建KEY_Task任务 \*/

140 xReturn = xTaskCreate((TaskFunction_t )KEY_Task, /\* 任务入口函数 \*/

141 (const char\* )"KEY_Task",/\* 任务名字 \*/

142 (uint16_t )512, /\* 任务栈大小 \*/

143 (void\* )NULL,/\* 任务入口函数参数 \*/

144 (UBaseType_t )3, /\* 任务的优先级 \*/

145 (TaskHandle_t\* )&KEY_Task_Handle);/\* 任务控制块指针 \*/

146 if (pdPASS == xReturn)

147 printf("创建KEY_Task任务成功!\n");

148

149 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

150

151 taskEXIT_CRITICAL(); //退出临界区

152 }

153

154

155

156 /\*

157 \* @ 函数名： LED_Task

158 \* @ 功能说明： LED_Task任务主体

159 \* @ 参数：

160 \* @ 返回值：无

161 \/

**162 static void LED_Task(void\* parameter)**

**163 {**

**164 EventBits_t r_event; /\* 定义一个事件接收变量 \*/**

**165 /\* 任务都是一个无限循环，不能返回 \*/**

**166 while (1) {**

**167 /\**

**168 \* 等待接收事件标志**

**169 \**

**170 \* 如果xClearOnExit设置为pdTRUE，那么在xEventGroupWaitBits()返回之前，**

**171 \* 如果满足等待条件（如果函数返回的原因不是超时），那么在事件组中设置**

**172 \* 的uxBitsToWaitFor中的任何位都将被清除。**

**173 \* 如果xClearOnExit设置为pdFALSE，**

**174 \* 则在调用xEventGroupWaitBits()时，不会更改事件组中设置的位。**

**175 \**

**176 \* xWaitForAllBits如果xWaitForAllBits设置为pdTRUE，则当uxBitsToWaitFor中**

**177 \* 的所有位都设置或指定的块时间到期时，xEventGroupWaitBits()才返回。**

**178 \* 如果xWaitForAllBits设置为pdFALSE，则当设置uxBitsToWaitFor中设置的任何**

**179 \* 一个位置1 或指定的块时间到期时，xEventGroupWaitBits()都会返回。**

**180 \* 阻塞时间由xTicksToWait参数指定。**

**181 \/**

**182 r_event = xEventGroupWaitBits(Event_Handle, /\* 事件对象句柄 \*/**

**183 KEY1_EVENT|KEY2_EVENT,/\* 接收任务感兴趣的事件 \*/**

**184 pdTRUE, /\* 退出时清除事件位 \*/**

**185 pdTRUE, /\* 满足感兴趣的所有事件 \*/**

**186 portMAX_DELAY);/\* 指定超时事件,一直等 \*/**

**187**

**188 if ((r_event & (KEY1_EVENT|KEY2_EVENT)) == (KEY1_EVENT|KEY2_EVENT)) {**

**189 /\* 如果接收完成并且正确 \*/**

**190 printf ( "KEY1与KEY2都按下\n");**

**191 LED1_TOGGLE; //LED1 反转**

**192 } else**

**193 printf ( "事件错误！\n");**

**194 }**

**195 }**

196

197 /\*

198 \* @ 函数名： KEY_Task

199 \* @ 功能说明： KEY_Task任务主体

200 \* @ 参数：

201 \* @ 返回值：无

202 \/

**203 static void KEY_Task(void\* parameter)**

**204 {**

**205 /\* 任务都是一个无限循环，不能返回 \*/**

**206 while (1) {//如果KEY2被按下**

**207 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**208 printf ( "KEY1被按下\n" );**

**209 /\* 触发一个事件1 \*/**

**210 xEventGroupSetBits(Event_Handle,KEY1_EVENT);**

**211 }**

**212 //如果KEY2被按下**

**213 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**214 printf ( "KEY2被按下\n" );**

**215 /\* 触发一个事件2 \*/**

**216 xEventGroupSetBits(Event_Handle,KEY2_EVENT);**

**217 }**

**218 vTaskDelay(20); //每20ms扫描一次**

**219 }**

**220 }**

221

222 /\*

223 \* @ 函数名： BSP_Init

224 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

225 \* @ 参数：

226 \* @ 返回值：无

227 \/

228 static void BSP_Init(void)

229 {

230 /\*

231 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

232 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

233 \* 都统一用这个优先级分组，千万不要再分组，切忌。

234 \*/

235 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

236

237 /\* LED 初始化 \*/

238 LED_GPIO_Config();

239

240 /\* 串口初始化 \*/

241 USART_Config();

242

243 /\* 按键初始化 \*/

244 Key_GPIO_Config();

245

246 }

247

248 /END OF FILE/

事件实验现象
~~~~~~

程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，按下开发版的KEY1按键发送
事件1，按下KEY2按键发送事件2；我们按下KEY1与KEY2试试，在串口调试助手中可以看到运行结果，并且当事件1与事件2都发生的时候，开发板的LED会进行翻转，具体见图18‑3。

|event004|

图18‑3事件标志组实验现象

.. |event002| image:: media\event002.png
   :width: 5.76806in
   :height: 0.81881in
.. |event003| image:: media\event003.png
   :width: 5.51693in
   :height: 5.00997in
.. |event004| image:: media\event004.png
   :width: 5.56643in
   :height: 3.09091in
