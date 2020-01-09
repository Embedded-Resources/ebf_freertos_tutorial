.. vim: syntax=rst

软件定时器
===========

软件定时器的基本概念
~~~~~~~~~~

定时器，是指从指定的时刻开始，经过一个指定时间，然后触发一个超时事件，用户可以自定义定时器的周期与频率。类似生活中的闹钟，我们可以设置闹钟每天什么时候响，还能设置响的次数，是响一次还是每天都响。

定时器有硬件定时器和软件定时器之分：

硬件定时器是芯片本身提供的定时功能。一般是由外部晶振提供给芯片输入时钟，芯片向软件模块提供一组配置寄存器，接受控制输入，到达设定时间值后芯片中断控制器产生时钟中断。硬件定时器的精度一般很高，可以达到纳秒级别，并且是中断触发方式。

软件定时器，软件定时器是由操作系统提供的一类系统接口，它构建在硬件定时器基础之上，使系统能够提供不受硬件定时器资源限制的定时器服务，它实现的功能与硬件定时器也是类似的。

使用硬件定时器时，每次在定时时间到达之后就会自动触发一个中断，用户在中断中处理信息；而使用软件定时器时，需要我们在创建软件定时器时指定时间到达后要调用的函数（也称超时函数/回调函数，为了统一，下文均用回调函数描述），在回调函数中处理信息。

注意：软件定时器回调函数的上下文是任务，下文所说的定时器均为软件定时器。

软件定时器在被创建之后，当经过设定的时钟计数值后会触发用户定义的回调函数。定时精度与系统时钟的周期有关。一般系统利用SysTick作为软件定时器的基础时钟，软件定时器的回调函数类似硬件的中断服务函数，所以，回调函数也要快进快出，而且回调函数中不能有任何阻塞任务运行的情况（软件定时器回调函数的上下文环
境是任务），比如vTaskDelay()以及其他能阻塞任务运行的函数，两次触发回调函数的时间间隔xTimerPeriodInTicks叫定时器的定时周期。

FreeRTOS操作系统提供软件定时器功能，软件定时器的使用相当于扩展了定时器的数量，允许创建更多的定时业务。FreeRTOS软件定时器功能上支持：

-  裁剪：能通过宏关闭软件定时器功能。

-  软件定时器创建。

-  软件定时器启动。

-  软件定时器停止。

-  软件定时器复位。

-  软件定时器删除。

FreeRTOS提供的软件定时器支持单次模式和周期模式，单次模式和周期模式的定时时间到之后都会调用软件定时器的回调函数，用户可以在回调函数中加入要执行的工程代码。

单次模式：当用户创建了定时器并启动了定时器后，定时时间到了，只执行一次回调函数之后就将该定时器删除，不再重新执行。

周期模式：这个定时器会按照设置的定时时间循环执行回调函数，直到用户将定时器删除，具体见图19‑1。

|softwa002|

图19‑1软件定时器的单次模式与周期模式

FreeRTOS 通过一个prvTimerTask任务（也叫守护任务Daemon）管理软定时器，它是在启动调度器时自动创建的，为了满足用户定时需求。prvTimerTask任务会在其执行期间检查用户启动的时间周期溢出的定时器，并调用其回调函数。只有设置 FreeRTOSConfig.h
中的宏定义configUSE_TIMERS设置为1 ，将相关代码编译进来，才能正常使用软件定时器相关功能。

软件定时器应用场景
~~~~~~~~~

在很多应用中，我们需要一些定时器任务，硬件定时器受硬件的限制，数量上不足以满足用户的实际需求，无法提供更多的定时器，那么可以采用软件定时器来完成，由软件定时器代替硬件定时器任务。但需要注意的是软件定时器的精度是无法和硬件定时器相比的，因为在软件定时器的定时过程中是极有可能被其他中断所打断，因为软件定
时器的执行上下文环境是任务。所以，软件定时器更适用于对时间精度要求不高的任务，一些辅助型的任务。

软件定时器的精度
~~~~~~~~

在操作系统中，通常软件定时器以系统节拍周期为计时单位。系统节拍是系统的心跳节拍，表示系统时钟的频率，就类似人的心跳，1s能跳动多少下，系统节拍配置为configTICK_RATE_HZ，该宏在FreeRTOSConfig.h中有定义，默认是1000。那么系统的时钟节拍周期就为1ms（1s跳动1000
下，每一下就为1ms）。软件定时器的所定时数值必须是这个节拍周期的整数倍，例如节拍周期是10ms，那么上层软件定时器定时数值只能是10ms，20ms，100ms等，而不能取值为15ms。由于节拍定义了系统中定时器能够分辨的精确度，系统可以根据实际系统CPU的处理能力和实时性需求设置合适的数值，系统节
拍周期的值越小，精度越高，但是系统开销也将越大，因为这代表在1秒中系统进入时钟中断的次数也就越多。

软件定时器的运作机制
~~~~~~~~~~

软件定时器是可选的系统资源，在创建定时器的时候会分配一块内存空间。当用户创建并启动一个软件定时器时，FreeRTOS会根据当前系统时间及用户设置的定时确定该定时器唤醒时间，并将该定时器控制块挂入软件定时器列表，FreeRTOS中采用两个定时器列表维护软件定时器，pxCurrentTimerList与
pxOverflowTimerList是列表指针，在初始化的时候分别指向xActiveTimerList1与xActiveTimerList2，具体见代码清单19‑1。

代码清单19‑1软件定时器用到的列表

1 PRIVILEGED_DATA static List_t xActiveTimerList1;

2 PRIVILEGED_DATA static List_t xActiveTimerList2;

3 PRIVILEGED_DATA static List_t \*pxCurrentTimerList;

4 PRIVILEGED_DATA static List_t \*pxOverflowTimerList;

pxCurrentTimerList：系统新创建并激活的定时器都会以超时时间升序的方式插入到pxCurrentTimerList列表中。系统在定时器任务中扫描pxCurrentTimerList中的第一个定时器，看是否已超时，若已经超时了则调用软件定时器回调函数。否则将定时器任务挂起，因为定时时间是
升序插入软件定时器列表的，列表中第一个定时器的定时时间都还没到的话，那后面的定时器定时时间自然没到。

pxOverflowTimerList列表是在软件定时器溢出的时候使用，作用与pxCurrentTimerList一致。

同时，FreeRTOS的软件定时器还有采用消息队列进行通信，利用“定时器命令队列”向软件定时器任务发送一些命令，任务在接收到命令就会去处理命令对应的程序，比如启动定时器，停止定时器等。假如定时器任务处于阻塞状态，我们又需要马上再添加一个软件定时器的话，就是采用这种消息队列命令的方式进行添加，才能唤醒
处于等待状态的定时器任务，并且在任务中将新添加的软件定时器添加到软件定时器列表中，所以，在定时器启动函数中，FreeRTOS是采用队列的方式发送一个消息给软件定时器任务，任务被唤醒从而执行接收到的命令。

例如：系统当前时间xTimeNow值为0，注意：xTimeNow其实是一个局部变量，是根据xTaskGetTickCount()函数获取的，实际它的值就是全局变量xTickCount的值，下文都采用它表示当前系统时间。在当前系统中已经创建并启动了1个定时器Timer1；系统继续运行，当系统的时间xT
imeNow为20的时候，用户创建并且启动一个定时时间为100的定时器Timer2，此时Timer2的溢出时间xTicksToWait就为定时时间+系统当前时间（100+20=120），然后将Timer2按xTicksToWait升序插入软件定时器列表中；假设当前系统时间xTimeNow为40的时候
，用户创建并且启动了一个定时时间为50的定时器Timer3，那么此时Timer3的溢出时间xTicksToWait就为40+50=90，同样安装xTicksToWait的数值升序插入软件定时器列表中，在定时器链表中插入过程具体见图19‑2。同理创建并且启动在已有的两个定时器中间的定时器也是一样的，具
体见图19‑3。

|softwa003|

图19‑2定时器链表示意图1

|softwa004|

图19‑3定时器链表示意图2

那么系统如何处理软件定时器列表？系统在不断运行，而xTimeNow（xTickCount）随着SysTick的触发一直在增长（每一次硬件定时器中断来临，xTimeNow变量会加1），在软件定时器任务运行的时候会获取下一个要唤醒的定时器，比较当前系统时间xTimeNow是否大于或等于下一个定时器唤醒时
间xTicksToWait，若大于则表示已经超时，定时器任务将会调用对应定时器的回调函数，否则将软件定时器任务挂起，直至下一个要唤醒的软件定时器时间到来或者接收到命令消息。以图19‑3为例，讲解软件定时器调用回调函数的过程，在创建定Timer1并且启动后，假如系统经过了50个tick，xTimeNo
w从0增长到50，与Timer1的xTicksToWait值相等，这时会触发与Timer1对应的回调函数，从而转到回调函数中执行用户代码，同时将Timer1从软件定时器列表删除，如果软件定时器是周期性的，那么系统会根据Timer1下一次唤醒时间重新将Timer1添加到软件定时器列表中，按照xTick
sToWait的升序进行排列。同理，在xTimeNow=40的时候创建的Timer3，在经过130个tick后（此时系统时间xTimeNow是40，130个tick就是系统时间xTimeNow为170的时候），与Timer3定时器对应的回调函数会被触发，接着将Timer3从软件定时器列表中删除，如果
是周期性的定时器，还会按照xTicksToWait升序重新添加到软件定时器列表中。

   使用软件定时器时候要注意以下几点：

-  软件定时器的回调函数中应快进快出，绝对不允许使用任何可能引软件定时器起任务挂起或者阻塞的API接口，在回调函数中也绝对不允许出现死循环。

-  软件定时器使用了系统的一个队列和一个任务资源，软件定时器任务的优先级默认为configTIMER_TASK_PRIORITY，为了更好响应，该优先级应设置为所有任务中最高的优先级。

-  创建单次软件定时器，该定时器超时执行完回调函数后，系统会自动删除该软件定时器，并回收资源。

-  定时器任务的栈大小默认为configTIMER_TASK_STACK_DEPTH个字节。

软件定时器控制块
~~~~~~~~

软件定时器虽然不属于内核资源，但是也是FreeRTOS核心组成部分，是一个可以裁剪的功能模块，同样在系统中由一个控制块管理其相关信息，软件定时器的控制块中包含没用过创建的软件定时器基本信息，在使用定时器前我们需要通过xTimerCreate()/xTimerCreateStatic()函数创建一个软
件定时器，在函数中，FreeRTOS将向系统管理的内存申请一块软件定时器控制块大小的内存用于保存定时器的信息，下面来看看软件定时器控制块的成员变量，具体见代码清单19‑2。

代码清单19‑2软件定时器控制块

1 typedefstruct tmrTimerControl {

2 const char \*pcTimerName; **(1)**

3 ListItem_t xTimerListItem; **(2)**

4 TickType_t xTimerPeriodInTicks;\ **(3)**

5 UBaseType_t uxAutoReload; **(4)**

6 void \*pvTimerID; **(5)**

7 TimerCallbackFunction_t pxCallbackFunction; **(6)**

8 #if( configUSE_TRACE_FACILITY == 1 )

9 UBaseType_t uxTimerNumber;

10 #endif

11

12 #if( ( configSUPPORT_STATIC_ALLOCATION == 1 )\\

13 && ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) )

14 uint8_t ucStaticallyAllocated; **(7)**

15 #endif

16 } xTIMER;

17

18 typedef xTIMER Timer_t;

代码清单19‑2\ **(1)**\ ：软件定时器名字，这个名字一般用于调试的，RTOS使用定时器是通过其句柄，并不是使用其名字。

代码清单19‑2\ **(2)**\ ：软件定时器列表项，用于插入定时器列表。

代码清单19‑2\ **(3)**\ ：软件定时器的周期，单位为系统节拍周期（即tick），pdMS_TO_TICKS()可以把时间单位从ms转换为系统节拍周期。

代码清单19‑2\ **(4)**\ ：软件定时器是否自动重置，如果该值为pdFalse，那么创建的软件定时器工作模式是单次模式，否则为周期模式。

代码清单19‑2\ **(5)**\ ：软件定时器ID，数字形式。该ID典型的用法是当一个回调函数分配给一个或者多个软件定时器时，在回调函数里面根据ID号来处理不同的软件定时器。

代码清单19‑2\ **(6)**\ ：软件定时器的回调函数，当定时时间到达的时候就会调用这个函数。

代码清单19‑2\ **(7)**\ ：标记定时器使用的内存，删除时判断是否需要释放内存。

软件定时器函数接口讲解
~~~~~~~~~~~

软件定时器的功能是在定时器任务（或者叫定时器守护任务）中实现的。软件定时器的很多API函数通过一个名字叫“定时器命令队列”的队列来给定时器守护任务发送命令。该定时器命令队列由RTOS内核提供，且应用程序不能够直接访问，其消息队列的长度由宏configTIMER_QUEUE_LENGTH定义，下面就讲
解一些常用的软件定时器函数接口。

软件定时器创建函数xTimerCreate()
^^^^^^^^^^^^^^^^^^^^^^^

软件定时器与FreeRTOS内核其他资源一样，需要创建才允许使用的，FreeRTOS为我们提供了两种创建方式，一种是动态创建软件定时器xTimerCreate()，另一种是静态创建方式xTimerCreateStatic()，因为创建过程基本差不多，所以在这里我们只讲解动态创建方式。

xTimerCreate()用于创建一个软件定时器，并返回一个句柄。要想使用该函数函数必须在头文件FreeRTOSConfig.h中把宏configUSE_TIMERS 和\ `configSUPPORT_DYNAMIC_ALLOCATION
<http://www.freertos.org/a00110.html#configSUPPORT_DYNAMIC_ALLOCATION>`__ 均定义为1（\ `configSUPPORT_DYNAMIC_ALLOCATION
<http://www.freertos.org/a00110.html#configSUPPORT_DYNAMIC_ALLOCATION>`__\ 在FreeRTOS.h中默认定义为1），并且需要把FreeRTOS/source/times.c 这个C文件添加到工程中。

每一个软件定时器只需要很少的RAM空间来保存其的状态。如果使用函数xTimeCreate()来创建一个软件定时器，那么需要的RAM是动态分配的。如果使用函数\ `xTimeCreateStatic
<http://www.freertos.org/xEventGroupCreateStatic.html>`__\ ()来创建一个事件组，那么需要的RAM是静态分配的

软件定时器在创建成功后是处于休眠状态的，可以使用\ `xTimerStart() <http://www.freertos.org/FreeRTOS-timers-xTimerStart.html>`__\ 、\ `xTimerReset()
<http://www.freertos.org/FreeRTOS-timers-xTimerReset.html>`__\ 、\ `xTimerStartFromISR() <http://www.freertos.org/FreeRTOS-timers-
xTimerStartFromISR.html>`__\ 、\ `xTimerResetFromISR() <http://www.freertos.org/FreeRTOS-timers-xTimerResetFromISR.html>`__\ 、 `xTimerChangePeriod()
<http://www.freertos.org/FreeRTOS-timers-xTimerChangePeriod.html>`__ 和\ `xTimerChangePeriodFromISR() <http://www.freertos.org/FreeRTOS-timers-
xTimerChangePeriodFromISR.html>`__\ 这些函数将其状态转换为活跃态。

xTimerCreate()函数源码具体见代码清单19‑3。

代码清单19‑3xTimerCreate()源码

1 #if( configSUPPORT_DYNAMIC_ALLOCATION == 1 )

2

3 TimerHandle_t xTimerCreate(const char \* const pcTimerName, **(1)**

4 const TickType_t xTimerPeriodInTicks,\ **(2)**

5 const UBaseType_t uxAutoReload, **(3)**

6 void \* const pvTimerID, **(4)**

7 TimerCallbackFunction_t pxCallbackFunction )\ **(5)**

8 {

9 Timer_t \*pxNewTimer;

10

11 /\* 为这个软件定时器申请一块内存 \*/

12 pxNewTimer = ( Timer_t \* ) pvPortMalloc( sizeof( Timer_t ) );\ **(6)**

13

14 if ( pxNewTimer != NULL ) {

15 /\* 内存申请成功，进行初始化软件定时器 \*/

16 prvInitialiseNewTimer( pcTimerName,

17 xTimerPeriodInTicks,

18 uxAutoReload,

19 pvTimerID,

20 pxCallbackFunction,

21 pxNewTimer ); **(7)**

22

23 #if( configSUPPORT_STATIC_ALLOCATION == 1 )

24 {

25 pxNewTimer->ucStaticallyAllocated = pdFALSE;

26 }

27 #endif

28 }

29

30 return pxNewTimer;

31 }

代码清单19‑3\ **(1)**\ ：软件定时器名字，文本形式，纯粹是为了调试，FreeRTOS使用定时器是通过其句柄，而不是使用其名字。

代码清单19‑3\ **(2)**\ ：软件定时器的周期，单位为系统节拍周期（即tick）。使用pdMS_TO_TICKS()可以把时间单位从ms转换为系统节拍周期。如果软件定时器的周期为100个tick，那么只需要简单的设置xTimerPeriod的值为100即可。如果软件定时器的周期为500ms
，那么xTimerPeriod应设置为pdMS_TO_TICKS(500)。宏pdMS_TO_TICKS()只有当\ `configTICK_RATE_HZ <http://www.freertos.org/a00110.html#configTICK_RATE_HZ>`__\
配置成小于或者等于1000HZ时才可以使用。

代码清单19‑3\ **(3)**\ ：如果uxAutoReload 设置为pdTRUE，那么软件定时器的工作模式就是周期模式，一直会以用户指定的xTimerPeriod周期去执行回调函数。如果uxAutoReload
设置为pdFALSE，那么软件定时器就在用户指定的xTimerPeriod周期下运行一次后就进入休眠态。

代码清单19‑3\ **(4)**\ ：软件定时器ID，数字形式。该ID典型的用法是当一个回调函数分配给一个或者多个软件定时器时，在回调函数里面根据ID号来处理不同的软件定时器。

代码清单19‑3\ **(5)**\ ：软件定时器的回调函数，当定时时间到达的时候就会调用这个函数，该函数需要用户自己实现。

代码清单19‑3\ **(6)**\ ：为这个软件定时器申请一块内存，大小为软件定时器控制块大小，用于保存该定时器的基本信息。

代码清单19‑3\ **(7)**\ ：调用prvInitialiseNewTimer()函数初始化一个新的软件定时器，该函数的源码具体见代码清单19‑4\ **(3)**\ ：。

代码清单19‑4 prvInitialiseNewTimer()源码

1 static void prvInitialiseNewTimer(const char \* const pcTimerName,

2 const TickType_t xTimerPeriodInTicks,

3 const UBaseType_t uxAutoReload,

4 void \* const pvTimerID,

5 TimerCallbackFunction_t pxCallbackFunction,

6 Timer_t \*pxNewTimer )

7 {

8 /\* 断言，判断定时器的周期是否大于0 \*/

9 configASSERT( ( xTimerPeriodInTicks > 0 ) ); **(1)**

10

11 if ( pxNewTimer != NULL ) {

12 /\* 初始化软件定时器列表与创建软件定时器消息队列 \*/

13 prvCheckForValidListAndQueue(); **(2)**

14

15 /\* 初始化软件定时信息，这些信息保存在软件定时器控制块中 \*/ **(3)**

16 pxNewTimer->pcTimerName = pcTimerName;

17 pxNewTimer->xTimerPeriodInTicks = xTimerPeriodInTicks;

18 pxNewTimer->uxAutoReload = uxAutoReload;

19 pxNewTimer->pvTimerID = pvTimerID;

20 pxNewTimer->pxCallbackFunction = pxCallbackFunction;

21 vListInitialiseItem( &( pxNewTimer->xTimerListItem ) ); **(4)**

22 traceTIMER_CREATE( pxNewTimer );

23 }

24 }

代码清单19‑4\ **(1)**\ ：断言，判断软件定时器的周期是否大于0，否则的话其他任务是没办法执行的，因为系统会一直执行软件定时器回调函数。

代码清单19‑4\ **(2)**\ ：在prvCheckForValidListAndQueue()函数中系统将初始化软件定时器列表与创建软件定时器消息队列，也叫“定时器命令队列”，因为在使用软件定时器的时候，用户是无法直接控制软件定时器的，必须通过“定时器命令队列”向软件定时器发送一个命令，软件
定时器任务被唤醒就去执行对应的命令操作。

代码清单19‑4\ **(3)**\ ：初始化软件定时基本信息，如定时器名称、回调周期、定时器ID与定时器回调函数等，这些信息保存在软件定时器控制块中，在操作软件定时器的时候，就需要用到这些信息。

代码清单19‑4\ **(4)**\ ：初始化定时器列表项。

软件定时器的创建很简单，需要用户根据自己需求指定相关信息即可，下面来看看xTimerCreate()函数使用实例，具体见代码清单19‑5加粗部分。

代码清单19‑5xTimerCreate()使用实例

1 static TimerHandle_t Swtmr1_Handle =NULL; /\* 软件定时器句柄 \*/

2 static TimerHandle_t Swtmr2_Handle =NULL; /\* 软件定时器句柄 \*/

3 /\* 周期模式的软件定时器1,定时器周期 1000(tick)*/

**4 Swtmr1_Handle=xTimerCreate((const char*)"AutoReloadTimer",**

**5 (TickType_t)1000,/\* 定时器周期 1000(tick) \*/**

**6 (UBaseType_t)pdTRUE,/\* 周期模式 \*/**

**7 (void\* )1,/\* 为每个计时器分配一个索引的唯一ID \*/**

**8 (TimerCallbackFunction_t)Swtmr1_Callback); /\* 回调函数 \*/**

9 if (Swtmr1_Handle != NULL)

10 {

11 /\*

12 \* xTicksToWait:如果在调用xTimerStart()时队列已满，则以tick为单位指定调用任务应保持

13 \* 在Blocked(阻塞)状态以等待start命令成功发送到timer命令队列的时间。

14 \* 如果在启动调度程序之前调用xTimerStart()，则忽略xTicksToWait。在这里设置等待时间为0.

15 \/

16 xTimerStart(Swtmr1_Handle,0); //开启周期定时器

17 }

18

19 /\* 单次模式的软件定时器2,定时器周期 5000(tick)*/

**20 Swtmr2_Handle=xTimerCreate((const char\* )"OneShotTimer",**

**21 (TickType_t)5000,/\* 定时器周期 5000(tick) \*/**

**22 (UBaseType_t )pdFALSE,/\* 单次模式 \*/**

**23 (void*)2,/\* 为每个计时器分配一个索引的唯一ID \*/**

**24 (TimerCallbackFunction_t)Swtmr2_Callback);**

25 if (Swtmr2_Handle != NULL)

26 {

27 xTimerStart(Swtmr2_Handle,0); //开启单次定时器

28 }

29

**30 static void Swtmr1_Callback(void\* parameter)**

31 {

32 /\* 软件定时器的回调函数，用户自己实现 \*/

33 }

34

**35 static void Swtmr2_Callback(void\* parameter)**

36 {

37 /\* 软件定时器的回调函数，用户自己实现 \*/

38 }

软件定时器启动函数
^^^^^^^^^

xTimerStart()
'''''''''''''

如果是认真看上面xTimerCreate()函数使用实例的同学应该就发现了，这个软件定时器启动函数xTimerStart()在上面的实例中有用到过，前一小节已经说明了，软件定时器在创建完成的时候是处于休眠状态的，需要用FreeRTOS的相关函数将软件定时器活动起来，而xTimerStart()函数就
是可以让处于休眠的定时器开始工作。

我们知道，在系统开始运行的时候，系统会帮我们自动创建一个软件定时器任务（prvTimerTask），在这个任务中，如果暂时没有运行中的定时器，任务会进入阻塞态等待命令，而我们的启动函数就是通过“定时器命令队列”向定时器任务发送一个启动命令，定时器任务获得命令就解除阻塞，然后执行启动软件定时器命令。下
面来看看xTimerStart()是怎么让定时器工作的吧，其源码具体见代码清单19‑6与代码清单19‑8。

代码清单19‑6xTimerStart()函数原型

1 #define xTimerStart( xTimer, xTicksToWait ) \\

2 xTimerGenericCommand( ( xTimer ), \\\ **(1)**

3 tmrCOMMAND_START, \\\ **(2)**

4 ( xTaskGetTickCount() ), \\\ **(3)**

5 NULL, \\\ **(4)**

6 ( xTicksToWait ) ) **(5)**

xTimerStart()函数就是一个宏定义，真正起作用的是xTimerGenericCommand()函数。

代码清单19‑6\ **(1)**\ ：要操作的软件定时器句柄。

代码清单19‑6\ **(2)**\ ：tmrCOMMAND_START是软件定时器启动命令，因为现在是要将软件定时器启动，该命令在timers.h中有定义。xCommandID参数可以指定多个命令，软件定时器操作支持的命令具体见代码清单19‑7。

代码清单19‑7软件定时器支持的命令

1 #define tmrCOMMAND_EXECUTE_CALLBACK_FROM_ISR ( ( BaseType_t ) -2 )

2 #define tmrCOMMAND_EXECUTE_CALLBACK ( ( BaseType_t ) -1 )

3 #define tmrCOMMAND_START_DONT_TRACE ( ( BaseType_t ) 0 )

4 #define tmrCOMMAND_START ( ( BaseType_t ) 1 )

5 #define tmrCOMMAND_RESET ( ( BaseType_t ) 2 )

6 #define tmrCOMMAND_STOP ( ( BaseType_t ) 3 )

7 #define tmrCOMMAND_CHANGE_PERIOD ( ( BaseType_t ) 4 )

8 #define tmrCOMMAND_DELETE ( ( BaseType_t ) 5 )

9

10 #define tmrFIRST_FROM_ISR_COMMAND ( ( BaseType_t ) 6 )

11 #define tmrCOMMAND_START_FROM_ISR ( ( BaseType_t ) 6 )

12 #define tmrCOMMAND_RESET_FROM_ISR ( ( BaseType_t ) 7 )

13 #define tmrCOMMAND_STOP_FROM_ISR ( ( BaseType_t ) 8 )

14 #define tmrCOMMAND_CHANGE_PERIOD_FROM_ISR ( ( BaseType_t ) 9 )

代码清单19‑6\ **(3)**\ ：获取当前系统时间。

代码清单19‑6\ **(4)**\ ：pxHigherPriorityTaskWoken为NULL，该参数在中断中发送命令才起作用。

代码清单19‑6\ **(5)**\ ：用户指定超时阻塞时间，单位为系统节拍周期(即tick)。调用xTimerStart()的任务将被锁定在阻塞态，在软件定时器把启动的命令成功发送到定时器命令队列之前。如果在FreeRTOS调度器开启之前调用xTimerStart()，形参将不起作用。

代码清单19‑8 xTimerGenericCommand()源码

1 BaseType_t xTimerGenericCommand( TimerHandle_t xTimer,

2 const BaseType_t xCommandID,

3 const TickType_t xOptionalValue,

4 BaseType_t \* const pxHigherPriorityTaskWoken,

5 const TickType_t xTicksToWait )

6 {

7 BaseType_t xReturn = pdFAIL;

8 DaemonTaskMessage_t xMessage;

9

10 configASSERT( xTimer );

11

12 /\* 发送命令给定时器任务 \*/

13 if ( xTimerQueue != NULL ) { **(1)**

14 /\* 要发送的命令信息，包含命令、

15 命令的数值（比如可以表示当前系统时间、要修改的定时器周期等）

16 以及要处理的软件定时器句柄 \*/

17 xMessage.xMessageID = xCommandID; **(2)**

18 xMessage.u.xTimerParameters.xMessageValue = xOptionalValue;

19 xMessage.u.xTimerParameters.pxTimer = ( Timer_t \* ) xTimer;

20

21 /\* 命令是在任务中发出的 \*/

22 if ( xCommandID < tmrFIRST_FROM_ISR_COMMAND ) { **(3)**

23 /\* 如果调度器已经运行了，就根据用户指定超时时间发送 \*/

24 if ( xTaskGetSchedulerState() == taskSCHEDULER_RUNNING ) {

25 xReturn = xQueueSendToBack( xTimerQueue,

26 &xMessage,

27 xTicksToWait ); **(4)**

28 } else {

29 /\* 如果调度器还未运行，发送就行了，不需要阻塞 \*/

30 xReturn = xQueueSendToBack( xTimerQueue,

31 &xMessage,

32 tmrNO_DELAY ); **(5)**

33 }

34 }

35 /\* 命令是在中断中发出的 \*/

36 else {

37 /\* 调用从中断向消息队列发送消息的函数 \*/

38 xReturn = xQueueSendToBackFromISR( xTimerQueue, **(6)**

39 &xMessage,

40 pxHigherPriorityTaskWoken );

41 }

42

43 traceTIMER_COMMAND_SEND( xTimer,

44 xCommandID,

45 xOptionalValue,

46 xReturn );

47 } else {

48 mtCOVERAGE_TEST_MARKER();

49 }

50

51 return xReturn;

52 }

代码清单19‑8\ **(1)**\ ：系统打算通过“定时器命令队列”发送命令给定时器任务，需要先判断一下“定时器命令队列”是否存在，只有存在队列才允许发送命令。

代码清单19‑8\ **(2)**\ ：要发送的命令基本信息，包括命令、命令的数值（比如可以表示当前系统时间、要修改的定时器周期等）以及要处理的软件定时器句柄等。

代码清单19‑8\ **(3)**\ ：根据用户指定的xCommandID参数，判断命令是在哪个上下文环境发出的，如果是在任务中发出的，则执行\ **(4)**\ 、\ **(5)**\ 代码，否则就执行\ **(6)**\ 。

代码清单19‑8\ **(4)**\ ：如果系统调度器已经运行了，就根据用户指定超时时间向“定时器命令队列”发送命令。

代码清单19‑8\ **(5)**\ ：如果调度器还未运行，用户指定的超时时间是无效的，发送就行了，不需要阻塞，tmrNO_DELAY的值为0。

代码清单19‑8\ **(6)**\ ：命令是在中断中发出的，调用从中断向消息队列发送消息的函数xQueueSendToBackFromISR()就行了。

软件定时器启动函数的使用很简单，在创建一个软件定时器完成后，就可以调用该函数启动定时器了，具体见代码清单19‑5。

xTimerStartFromISR()
''''''''''''''''''''

当然除在任务启动软件定时器之外，还有在中断中启动软件定时器的函数xTimerStartFromISR()。xTimerStartFromISR()是函数xTimerStart()的中断版本，用于启动一个先前由函数\ `xTimerCreate()
<http://www.freertos.org/FreeRTOS-timers-xTimerCreate.html>`__ /xTimerCreateStatic()创建的软件定时器。该函数的具体说明见表19‑1，使用实例具体见代码清单19‑9。

表19‑1 xTimerStartFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | #d
     - fine                  | xTimerStartFromISR( xTimer, p xHigherPriorityTaskWoken )  xTimerGenericCommand( ( xTimer ), tm rCOMMAND_START_FROM_ISR,  ( xT
       askGetTickCountFromISR() ),  ( p xHigherPriorityTaskWoken ), 0U )
     - |

   * - **功能**     |
     - 中                     | 断中启动一个软件定时器。 |
     - |

   * - **形参**     |
     - Timer                   |
     - 件定时器句柄。         |

   * -
     - p xHigherPriorityTaskWoken
     - 定时器守护任务的大部     | 分时间都在阻塞态等待定时 | 器命令队列的命令。调用函 | 数xTimerStartFromISR()将 | 会往定时器的命令队列发送 | 一个启动命令，这很有可能 | 会将定时器任务从阻塞态移 | 除。如果调用函数xTimerSt |
       artFromISR()让定时器任务 | 脱离阻塞态，且定时器守护 | 任务的优先级大于或者等于 | 当前被中断的任务的优先级 | ，那么pxHigherPriorityT  | askWoken的值会在函数xTim | erStartFromISR()内部设置 | 为pdTRUE，然后在中断退出
       | 之前执行一次上下文切换。 |

   * - **返回值**   | 如
     - | 启动命令无法成功地发送到 | 定时器命令队列则返回pdF  | AILE，成功发送则返回pdPA | SS。软件定时器成功发送的 | 命令是否真正的被执行也还 | 要看定时器守护任务的优先 | 级，其优先级由宏configT  | IMER_TASK_PRIORITY定义。 |
     - |


         |


代码清单19‑9xTimerStartFromISR()函数应用举例

1 /\* 这个方案假定软件定时器xBacklightTimer已经创建，

2 定时周期为5s，执行次数为一次，即定时时间到了之后

3 就进入休眠态。

4 程序说明：当按键按下，打开液晶背光，启动软件定时器，

5 5s时间到，关掉液晶背光*/

6

7 /\* 软件定时器回调函数 \*/

8 void vBacklightTimerCallback( TimerHandle_t pxTimer )

9 {

10 /\* 关掉液晶背光 \*/

11 vSetBacklightState( BACKLIGHT_OFF );

12 }

13

14

15 /\* 按键中断服务程序 \*/

16 void vKeyPressEventInterruptHandler( void )

17 {

18 BaseType_t xHigherPriorityTaskWoken = pdFALSE;

19

20 /\* 确保液晶背光已经打开 \*/

21 vSetBacklightState( BACKLIGHT_ON );

22

23 /\* 启动软件定时器 \*/

**24 if ( xTimerStartFromISR( xBacklightTimer,**

**25 &xHigherPriorityTaskWoken ) != pdPASS ) {**

**26 /\* 软件定时器开启命令没有成功执行 \*/**

**27 }**

28

29 /\* ...执行其他的按键相关的功能代码 \*/

30

**31 if ( xHigherPriorityTaskWoken != pdFALSE ) {**

**32 /\* 执行上下文切换 \*/**

33 }

34 }

软件定时器停止函数
^^^^^^^^^

xTimerStop()
''''''''''''

xTimerStop() 用于停止一个已经启动的软件定时器，该函数的实现也是通过“定时器命令队列”发送一个停止命令给软件定时器任务，从而唤醒软件定时器任务去将定时器停止。要想使函数xTimerStop()必须在头文件FreeRTOSConfig.h中把宏configUSE_TIMERS定义为1。该函
数的具体说明见表19‑2。

表19‑2xTimerStop()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t xTimerStop(   | TimerHandle_t xTimer, TickType_t xBlockTime );
     - |

   * - **功能**     |
     - 止一个软件             | 定时器，让其进入休眠态。 |
     - |

   * - **形参**     |
     - Timer                   |
     - 件定时器句柄。         |

   * -
     - xBlockTime
     - 用户指定超时             | 时间，单位为系统节拍周期 | (即tick)。如果在FreeRTOS | 调度器开启之前调用xTimer | Stop()，形参将不起作用。 |

   * - **返回值**   | 如
     - 启动命令在超时       | 时间之前无法成功地发送到 | 定时器命令队列则返回pdF  | AILE，成功发送则返回pdPA | SS。软件定时器成功发送的 | 命令是否真正的被执行也还 | 要看定时器守护任务的优先 | 级，其优先级由宏configT  |
       IMER_TASK_PRIORITY定义。 |
     - |
       |
         |
           |
        |
       |
       |
           |
                |


软件定时器停止函数的使用实例很简单，在使用该函数前请确认定时器已经开启，具体见代码清单19‑10加粗部分。

代码清单19‑10xTimerStop()使用实例

1 static TimerHandle_t Swtmr1_Handle =NULL; /\* 软件定时器句柄 \*/

2

3 /\* 周期模式的软件定时器1,定时器周期 1000(tick)*/

4 Swtmr1_Handle=xTimerCreate((const char\* )"AutoReloadTimer",

5 (TickType_t )1000,/\* 定时器周期 1000(tick) \*/

6 (UBaseType_t )pdTRUE,/\* 周期模式 \*/

7 (void*)1,/\* 为每个计时器分配一个索引的唯一ID \*/

8 (TimerCallbackFunction_t)Swtmr1_Callback); /\* 回调函数 \*/

9 if (Swtmr1_Handle != NULL)

10 {

11 /\*

12 \* xTicksToWait:如果在调用xTimerStart()时队列已满，则以tick为单位指定调用任务应保持

13 \* 在Blocked(阻塞)状态以等待start命令成功发送到timer命令队列的时间。

14 \* 如果在启动调度程序之前调用xTimerStart()，则忽略xTicksToWait。在这里设置等待时间为0.

15 \/

**16 xTimerStart(Swtmr1_Handle,0); //开启周期定时器**

17 }

18

19 static void test_task(void\* parameter)

20 {

21 while (1) {

22 /\* 用户自己实现任务代码 \*/

**23 xTimerStop(Swtmr1_Handle,0); //停止定时器**

24 }

25

26 }

xTimerStopFromISR()
'''''''''''''''''''

xTimerStopFromISR()是函数xTimerStop()的中断版本，用于停止一个正在运行的软件定时器，让其进入休眠态，实现过程也是通过“定时器命令队列”向软件定时器任务发送停止命令。该函数的具体说明见表19‑3，应用举例见代码清单19‑11加粗部分。

表19‑3xTimerStopFromISR()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | Ba
     - eType_t               | xTimerS topFromISR(TimerHandle_t xTimer,  BaseType_t \*pxH igherPriorityTaskWoken);
     - |

   * - **功能**     |
     - 中断中停止一个软件     | 定时器，让其进入休眠态。 |
     - |
       |

   * - **形参**     |
     - Timer                   |
     - 件定时器句柄。         |

   * -
     - p xHigherPriorityTaskWoken
     - 定时器守护任务的大       | 部分时间都在阻塞态等待定 | 时器命令队列的命令。调用 | 函数xTimerStopFromISR()  | 将会往定时器的命令队列发 | 送一个停止命令，这很有可 | 能会将定时器任务从阻塞态 | 移除。如果调用函数xTime  |
       rStopFromISR()让定时器任 | 务脱离阻塞态，且定时器守 | 护任务的优先级大于或者等 | 于当前被中断的任务的优先 | 级，那么pxHigherPriority | TaskWoken的值会在函数xTi | merStopFromISR()内部设置 | 为pdTRUE，然后在中断退出
       | 之前执行一次上下文切换。 |

   * - **返回值**   | 如
     - 停止命令在超时       | 时间之前无法成功地发送到 | 定时器命令队列则返回pdF  | AILE，成功发送则返回pdPA | SS。软件定时器成功发送的 | 命令是否真正的被执行也还 | 要看定时器守护任务的优先 | 级，其优先级由宏configT  |
       IMER_TASK_PRIORITY定义。 |
     - |
       |
         |
           |
        |
       |
       |
           |
                |


代码清单19‑11xTimerStopFromISR()函数应用举例

1 /\* 这个方案假定软件定时器xTimer已经创建且启动。

2 当中断发生时，停止软件定时器 \*/

3

4 /\* 停止软件定时器的中断服务函数*/

5 void vAnExampleInterruptServiceRoutine( void )

6 {

7 BaseType_t xHigherPriorityTaskWoken = pdFALSE;

8

**9 if (xTimerStopFromISR(xTimer,&xHigherPriorityTaskWoken)!=pdPASS ) {**

10 /\* 软件定时器停止命令没有成功执行 \*/

11 }

12

13

**14 if ( xHigherPriorityTaskWoken != pdFALSE ) {**

15 /\* 执行上下文切换 \*/

16 }

17 }

软件定时器任务
^^^^^^^

我们知道，软件定时器回调函数运行的上下文环境是任务，那么软件定时器任务是在干什么的呢？如何创建的呢？下面跟我一步步来分析软件定时器的工作过程。

软件定时器任务是在系统开始调度（vTaskStartScheduler()函数）的时候就被创建的，前提是将宏定义configUSE_TIMERS开启，具体见代码清单19‑12加粗部分，在xTimerCreateTimerTask()函数里面就是创建了一个软件定时器任务，就跟我们创建任务一样，支持动态
与静态创建，我们暂时看动态创建的即可，具体见代码清单19‑13加粗部分。

代码清单19‑12 vTaskStartScheduler()函数里面的创建定时器函数（已删减）

1 void vTaskStartScheduler( void )

2 {

3 #if ( configUSE_TIMERS == 1 )

4 {

5 if ( xReturn == pdPASS )

6 {

**7 xReturn = xTimerCreateTimerTask();**

8 } else

9 {

10 mtCOVERAGE_TEST_MARKER();

11 }

12 }

13 #endif/\* configUSE_TIMERS \*/

14

15 }

代码清单19‑13 xTimerCreateTimerTask()源码

1 BaseType_t xTimerCreateTimerTask( void )

2 {

3 BaseType_t xReturn = pdFAIL;

4

5 prvCheckForValidListAndQueue();

6

7 if ( xTimerQueue != NULL ) {

8 #if( configSUPPORT_STATIC_ALLOCATION == 1 ) /\* 静态创建任务 \*/

9 {

10 StaticTask_t \*pxTimerTaskTCBBuffer = NULL;

11 StackType_t \*pxTimerTaskStackBuffer = NULL;

12 uint32_t ulTimerTaskStackSize;

13

14 vApplicationGetTimerTaskMemory( &pxTimerTaskTCBBuffer,

15 &pxTimerTaskStackBuffer,

16 &ulTimerTaskStackSize );

17 xTimerTaskHandle = xTaskCreateStatic(prvTimerTask,

18 "Tmr Svc",

19 ulTimerTaskStackSize,

20 NULL,

21 ( ( UBaseType_t ) configTIMER_TASK_PRIORITY ) \| portPRIVILEGE_BIT,

22 pxTimerTaskStackBuffer,

23 pxTimerTaskTCBBuffer );

24

25 if ( xTimerTaskHandle != NULL )

26 {

27 xReturn = pdPASS;

28 }

29 }

30 #else /\* 动态创建任务 \*/

31 {

**32 xReturn = xTaskCreate(prvTimerTask,**

**33 "Tmr Svc",**

**34 configTIMER_TASK_STACK_DEPTH,**

**35 NULL,**

**36 ( ( UBaseType_t ) configTIMER_TASK_PRIORITY ) \| portPRIVILEGE_BIT,**

**37 &xTimerTaskHandle ); (1)**

**38 }**

39 #endif

40 } else {

41 mtCOVERAGE_TEST_MARKER();

42 }

43

44 configASSERT( xReturn );

45 return xReturn;

46 }

代码清单19‑13\ **(1)**\
：系统调用xTaskCreate()函数创建了一个软件定时器任务，任务的入口函数是prvTimerTask，任务的优先级是configTIMER_TASK_PRIORITY，那么我们就去软件定时器任务函数prvTimerTask()中看看任务在做什么东西，具体见代码清单19‑14。

代码清单19‑14prvTimerTask()源码（已删减）

1 static void prvTimerTask( void \*pvParameters )

2 {

3 TickType_t xNextExpireTime;

4 BaseType_t xListWasEmpty;

5

6 ( void ) pvParameters;

7

8 for ( ;; ) {

9 /\* 获取下一个要到期的软件定时器的时间 \*/

10 xNextExpireTime = prvGetNextExpireTime( &xListWasEmpty );\ **(1)**

11

12 /\* 处理定时器或者将任务阻塞到下一个到期的软件定时器时间 \*/

13 prvProcessTimerOrBlockTask( xNextExpireTime, xListWasEmpty );\ **(2)**

14

15 /\* 读取“定时器命令队列”，处理相应命令 \*/

16 prvProcessReceivedCommands(); **(3)**

17 }

18 }

软件定时器任务的处理很简单，如果当前有软件定时器在运行，那么它大部分的时间都在等待定时器到期时间的到来，或者在等待对软件定时器操作的命令，而如果没有软件定时器在运行，那定时器任务的绝大部分时间都在阻塞中等待定时器的操作命令。

代码清单19‑14\ **(1)**\ ：获取下一个要到期的软件定时器的时间，因为软件定时器是由定时器列表维护的，并且按照到期的时间进行升序排列，只需获取软件定时器列表中的第一个定时器到期时间就是下一个要到期的时间。

代码清单19‑14\ **(2)**\ ：处理定时器或者将任务阻塞到下一个到期的软件定时器时间，因为系统时间节拍随着系统的运行可能会溢出，那么就需要处理溢出的情况，如果没有溢出，那么就等待下一个定时器到期时间的到来。该函数每次调用都会记录节拍值，下一次调用，通过比较相邻两次调用的值判断节拍计数器是否
溢出过。当节拍计数器溢出，需要处理掉当前定时器列表上的定时器（因为这条定时器列表上的定时器都已经溢出了），然后切换定时器列表。

软件定时器是一个任务，在下一个定时器到了之前的这段时间，系统要把任务状态转移为阻塞态，让其他的任务能正常运行，这样子就使得系统的资源能充分利用，prvProcessTimerOrBlockTask()源码具体见代码清单19‑15。

代码清单19‑15prvProcessTimerOrBlockTask()源码

1 static void prvProcessTimerOrBlockTask( const TickType_t xNextExpireTime,

2 BaseType_t xListWasEmpty )

3 {

4 TickType_t xTimeNow;

5 BaseType_t xTimerListsWereSwitched;

6

7 vTaskSuspendAll(); **(1)**

8 {

9 // 获取当前系统时间节拍并判断系统节拍计数是否溢出

10 // 如果是，那么就处理当前列表上的定时器，并切换定时器列表

11 xTimeNow = prvSampleTimeNow( &xTimerListsWereSwitched );\ **(2)**

12

13 // 系统节拍计数器没有溢出

14 if ( xTimerListsWereSwitched == pdFALSE ) { **(3)**

15 // 判断是否有定时器是否到期，

16 //定时器列表非空并且定时器的时间已比当前时间小，说明定时器到期了

17 if ((xListWasEmpty == pdFALSE )&&(xNextExpireTime <= xTimeNow )){**(4)**

18 // 恢复调度器

19 ( void ) xTaskResumeAll();

20 //执行相应定时器的回调函数

21 // 对于需要自动重载的定时器，更新下一次溢出时间，插回列表

22 prvProcessExpiredTimer( xNextExpireTime, xTimeNow );

23 } else {

24 // 当前定时器列表中没有定时器

25 if ( xListWasEmpty != pdFALSE ) { **(5)**

26 //发生这种情况的可能是系统节拍计数器溢出了，

27 //定时器被添加到溢出列表中，所以判断定时器溢出列表上是否有定时器

28 xListWasEmpty = listLIST_IS_EMPTY( pxOverflowTimerList );

29 }

30

31 // 定时器定时时间还没到，将当前任务挂起，

32 // 直到定时器到期才唤醒或者收到命令的时候唤醒

33 vQueueWaitForMessageRestricted( xTimerQueue,

34 ( xNextExpireTime - xTimeNow ),

35 xListWasEmpty ); **(6)**

36

37 // 恢复调度器

38 if ( xTaskResumeAll() == pdFALSE ) {

39 // 进行任务切换

40 portYIELD_WITHIN_API(); **(7)**

41 } else {

42 mtCOVERAGE_TEST_MARKER();

43 }

44 }

45 } else {

46 ( void ) xTaskResumeAll();

47 }

48 }

49 }

代码清单19‑15\ **(1)**\ ：挂起调度器。接下来的操作会对定时器列表进行操作，系统不希望别的任务来操作定时器列表，所以暂时让定时器任务独享CPU使用权，在此期间不进行任务切换。

代码清单19‑15\ **(2)**\ ：获取当前系统时间节拍并判断系统节拍计数是否溢出，如果已经溢出了，那么就处理当前列表上的定时器，并切换定时器列表，prvSampleTimeNow()函数就实现这些功能，其源码具体见代码清单19‑16。

代码清单19‑16prvSampleTimeNow()源码

1 static TickType_t prvSampleTimeNow( BaseType_t \* const pxTimerListsWereSwitched )

2 {

3 TickType_t xTimeNow;

4 // 定义一个静态变量记录上一次调用时系统时间节拍值

5 PRIVILEGED_DATA static TickType_t xLastTime = ( TickType_t ) 0U;\ **(1)**

6

7 //获取当前系统时间节拍

8 xTimeNow = xTaskGetTickCount(); **(2)**

9

10 //判断是否溢出了，

11 //当前系统时间节拍比上一次调用时间节拍的值小，这种情况是溢出的情况

12 if ( xTimeNow < xLastTime ) { **(3)**

13 // 发生溢出，处理当前定时器列表上所有定时器并切换定时器列表

14 prvSwitchTimerLists();

15 \*pxTimerListsWereSwitched = pdTRUE;

16 } else {

17 \*pxTimerListsWereSwitched = pdFALSE;

18 }

19 // 更新本次系统时间节拍

20 xLastTime = xTimeNow; **(4)**

21

22 return xTimeNow; **(5)**

23 }

代码清单19‑16\ **(1)**\ ：定义一个静态变量，记录上一次调用时系统时间节拍的值。

代码清单19‑16\ **(2)**\ ：获取当前系统时间节拍值。

代码清单19‑16\ **(3)**\ ：判断是系统节拍计数器否溢出了，当前系统时间节拍比上一次调用时间节拍的值小，这种情况是溢出的情况。而如果发生了溢出，系统就要处理当前定时器列表上所有定时器并切将当前时器列表的定时器切换到定时器溢出列表中，因为软件定时器由两个列表维护，并且标记一下定时器列表已经
切换了，pxTimerListsWereSwitched的值等于pdTRUE。

代码清单19‑16\ **(4)**\ ：更新本次系统时间节拍的值。

代码清单19‑16\ **(5)**\ ：返回当前系统时间节拍。

代码清单19‑15\ **(3)**\ ：如果系统节拍计数器没有溢出。

代码清单19‑15\ **(4)**\ ：判断是否有定时器是否到期可以触发回调函数，如果定时器列表非空并且定时器的时间已比当前时间小，说明定时器到期了，系统可用恢复调度器，并且执行相应到期的定时器回调函数，对于需要自动重载的定时器，更新下一次溢出时间，然后插回定时器列表中，这些操作均在prvProc
essExpiredTimer()函数中执行。

代码清单19‑15\ **(5)**\ ：定时器没有到期，后看看当前定时器列表中没有定时器，如果没有，那么发生这种情况的可能是系统节拍计数器溢出了，定时器被添加到溢出列表中，所以判断一下定时器溢出列表上是否有定时器。

代码清单19‑15\ **(6)**\ ：定时器定时时间还没到，将当前的定时器任务阻塞，直到定时器到期才唤醒或者收到命令的时候唤醒。FreeRTOS采用获取“定时器命令队列”的命令的方式阻塞当前任务，阻塞时间为下一个定时器到期时间节拍减去当前系统时间节拍，为什么呢？因为获取消息队列的时候，没有消息会
将任务阻塞，时间由用户指定，这样子一来，既不会错过定时器的到期时间，也不会错过操作定时器的命令。

代码清单19‑15\ **(7)**\ ：恢复调度器，看看是否有任务需要切换，如果有则进行任务切换。

以上就是软件定时器任务中的prvProcessTimerOrBlockTask()函数执行的代码，这样子看来，软件定时器任务大多数时间都处于阻塞状态的，而且一般在FreeRTOS中，软件定时器任务一般设置为所有任务中最高优先级，这样一来，定时器的时间一到，就会马上到定时器任务中执行对应的回调函数。

代码清单19‑14\ **(3)**\ ：读取“定时器命令队列”，处理相应命令，前面我们已经讲解一下定时器的函数是通过发送命令去控制定时器的，而定时器任务就需要有一个接收命令并且处理的函数，prvProcessReceivedCommands()源码具体见代码清单19‑17。

代码清单19‑17 prvProcessReceivedCommands()源码（已删减）

1 static void prvProcessReceivedCommands( void )

2 {

3 DaemonTaskMessage_t xMessage;

4 Timer_t \*pxTimer;

5 BaseType_t xTimerListsWereSwitched, xResult;

6 TickType_t xTimeNow;

7

8 while ( xQueueReceive( xTimerQueue, &xMessage, tmrNO_DELAY ) != pdFAIL ) {

9 /\* 判断定时器命令是否有效 \*/

10 if ( xMessage.xMessageID >= ( BaseType_t ) 0 ) {

11

12 /\* 获取定时器消息，获取命令指定处理的定时器，*/

13 pxTimer = xMessage.u.xTimerParameters.pxTimer;

14

15 if ( listIS_CONTAINED_WITHIN( NULL,

16 &( pxTimer->xTimerListItem ) ) == pdFALSE ) {

17 /\* 如果定时器在列表中，不管三七二十一，将定时器移除 \*/

18 ( void ) uxListRemove( &( pxTimer->xTimerListItem ) );

19 } else {

20 mtCOVERAGE_TEST_MARKER();

21 }

22

23 traceTIMER_COMMAND_RECEIVED( pxTimer,

24 xMessage.xMessageID,

25 xMessage.u.xTimerParameters.xMessageValue );

26

27 // 判断节拍计数器是否溢出过，如果有就处理并切换定时器列表

28 // 因为下面的操作可能有新定时器项插入确保定时器列表对应

29 xTimeNow = prvSampleTimeNow( &xTimerListsWereSwitched );

30

31 switch ( xMessage.xMessageID ) {

32 case tmrCOMMAND_START :

33 case tmrCOMMAND_START_FROM_ISR :

34 case tmrCOMMAND_RESET :

35 case tmrCOMMAND_RESET_FROM_ISR :

36 case tmrCOMMAND_START_DONT_TRACE :

37 // 以上的命令都是让定时器启动

38 // 求出定时器到期时间并插入到定时器列表中

39 if ( prvInsertTimerInActiveList( pxTimer,

40 xMessage.u.xTimerParameters.xMessageValue

41 + pxTimer->xTimerPeriodInTicks,

42 xTimeNow,

43 xMessage.u.xTimerParameters.xMessageValue )

44 != pdFALSE ) {

45 // 该定时器已经溢出赶紧执行其回调函数

46 pxTimer->pxCallbackFunction( ( TimerHandle_t ) pxTimer );

47 traceTIMER_EXPIRED( pxTimer );

48

49 // 如果定时器是重载定时器，就重新启动

50 if ( pxTimer->uxAutoReload == ( UBaseType_t ) pdTRUE ) {

51 xResult = xTimerGenericCommand( pxTimer,

52 tmrCOMMAND_START_DONT_TRACE,

53 xMessage.u.xTimerParameters.xMessageValue

54 + pxTimer->xTimerPeriodInTicks,

55 NULL,

56 tmrNO_DELAY );

57 configASSERT( xResult );

58 ( void ) xResult;

59 } else {

60 mtCOVERAGE_TEST_MARKER();

61 }

62 } else {

63 mtCOVERAGE_TEST_MARKER();

64 }

65 break;

66

67 case tmrCOMMAND_STOP :

68 case tmrCOMMAND_STOP_FROM_ISR :

69 // 如果命令是停止定时器，那就将定时器移除，

70 // 在开始的时候已经从定时器列表移除，

71 // 此处就不需要做其他操作

72 break;

73

74 case tmrCOMMAND_CHANGE_PERIOD :

75 case tmrCOMMAND_CHANGE_PERIOD_FROM_ISR :

76 // 更新定时器配置

77 pxTimer->xTimerPeriodInTicks

78 = xMessage.u.xTimerParameters.xMessageValue;

79 configASSERT( ( pxTimer->xTimerPeriodInTicks > 0 ) );

80

81 // 插入到定时器列表，也重新启动了定时器

82 ( void ) prvInsertTimerInActiveList( pxTimer,

83 ( xTimeNow + pxTimer->xTimerPeriodInTicks ),

84 xTimeNow,

85 xTimeNow );

86 break;

87

88 case tmrCOMMAND_DELETE :

89 // 删除定时器

90 // 判断定时器内存是否需要释放（动态的释放）

91 #if( ( configSUPPORT_DYNAMIC_ALLOCATION == 1 )\\

92 && ( configSUPPORT_STATIC_ALLOCATION == 0 ) )

93 {

94 /\* 动态释放内存*/

95 vPortFree( pxTimer );

96 }

97break;

98

99default :

100/\* Don't expect to get here.
\*/

101break;

102 }

103 }

104 }

105}

其实处理这些软件定时器命令是很简单的，当任务获取到命令消息的时候，会先移除对应的定时器，无论是什么原因，然后就根据命令去处理对应定时器的操作即可，具体见代码清单19‑17的源码注释即可。

软件定时器删除函数xTimerDelete()
^^^^^^^^^^^^^^^^^^^^^^^

xTimerDelete()用于删除一个已经被创建成功的软件定时器，删除之后就无法使用该定时器，并且定时器相应的资源也会被系统回收释放。要想使函数xTimerDelete()必须在头文件FreeRTOSConfig.h中把宏configUSE_TIMERS定义为1，该函数的具体说明见表19‑4。

表19‑4xTimerDelete()函数说明

.. list-table::
   :widths: 33 33 33
   :header-rows: 0


   * - **函数原型** | #d
     - fine xTimerDelete(    | xTimer, xTicksToWait )  xTimerGenericCommand( ( xTimer ),  tmrCOMMAND_DELETE,  0U, NULL, ( xTicksToWait ) )
     - |

   * - **功能**     |
     - 除一个已经             | 被创建成功的软件定时器。 |
     - |

   * - **形参**     |
     - Timer                   |
     - 件定时器句柄。         |

   * -
     - xBlockTime
     - 用户指定的超时时间       | ，单位为系统节拍周期(即t | ick)，如果在FreeRTOS调度 | 器开启之前调用xTimerDele | te()，该形参将不起作用。 |

   * - **返回值**   | 如
     - 删除命令在超时时间   | 之前无法成功地发送到定时 | 器命令队列则返回pdFAILE  | ，成功发送则返回pdPASS。 |
     - |
         |
             |
            |


从软件定时器删除函数xTimerDelete()的原型可以看出，删除一个软件定时器也是在软件定时器任务中删除，调用xTimerDelete()将删除软件定时器的命令发送给软件定时器任务，软件定时器任务在接收到删除的命令之后就进行删除操作，该函数的使用方法很简单，具体见代码清单19‑18加粗部分。

代码清单19‑18xTimerDelete()使用实例

1 static TimerHandle_t Swtmr1_Handle =NULL; /\* 软件定时器句柄 \*/

2

3 /\* 周期模式的软件定时器1,定时器周期 1000(tick)*/

4 Swtmr1_Handle=xTimerCreate((const char\* )"AutoReloadTimer",

5 (TickType_t )1000,/\* 定时器周期 1000(tick) \*/

6 (UBaseType_t)pdTRUE,/\* 周期模式 \*/

7 (void\* )1,/\* 为每个计时器分配一个索引的唯一ID \*/

8 (TimerCallbackFunction_t)Swtmr1_Callback); /\* 回调函数 \*/

9 if (Swtmr1_Handle != NULL)

10 {

11 /\*

12 \* xTicksToWait:如果在调用xTimerStart()时队列已满，则以tick为单位指定调用任务应保持

13 \* 在Blocked(阻塞)状态以等待start命令成功发送到timer命令队列的时间。

14 \* 如果在启动调度程序之前调用xTimerStart()，则忽略xTicksToWait。在这里设置等待时间为0.

15 \/

16 xTimerStart(Swtmr1_Handle,0); //开启周期定时器

17 }

18

19 static void test_task(void\* parameter)

20 {

21 while (1) {

22 /\* 用户自己实现任务代码 \*/

**23 xTimerDelete(Swtmr1_Handle,0); //删除软件定时器**

24 }

25 }

软件定时器实验
~~~~~~~

软件定时器实验是在FreeRTOS中创建了两个软件定时器，其中一个软件定时器是单次模式，5000个tick调用一次回调函数，另一个软件定时器是周期模式，1000个tick调用一次回调函数，在回调函数中输出相关信息，具体见代码清单19‑19加粗部分。

代码清单19‑19软件定时器实验

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS V9.0.0 + STM32 软件定时器

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

31 /\* 任务句柄 \/

32 /\*

33 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

34 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

35 \* 这个句柄可以为NULL。

36 \*/

37 static TaskHandle_t AppTaskCreate_Handle = NULL;/\* 创建任务句柄 \*/

38

39 /\* 内核对象句柄 \/

40 /\*

41 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

42 \* 对象，必须先创建，创建成功之后会返回一个相应的句柄。实际上就是一个指针，后续我

43 \* 们就可以通过这个句柄操作这些内核对象。

44 \*

45 \*

46 内核对象说白了就是一种全局的数据结构，通过这些数据结构我们可以实现任务间的通信，

47 \* 任务间的事件同步等各种功能。至于这些功能的实现我们是通过调用这些内核对象的函数

48 \* 来完成的

49 \*

50 \*/

**51 static TimerHandle_t Swtmr1_Handle =NULL; /\* 软件定时器句柄 \*/**

**52 static TimerHandle_t Swtmr2_Handle =NULL; /\* 软件定时器句柄 \*/**

53 /\* 全局变量声明 \/

54 /\*

55 \* 当我们在写应用程序的时候，可能需要用到一些全局变量。

56 \*/

57 static uint32_t TmrCb_Count1 = 0; /\* 记录软件定时器1回调函数执行次数 \*/

58 static uint32_t TmrCb_Count2 = 0; /\* 记录软件定时器2回调函数执行次数 \*/

59

60 /\* 宏定义 \/

61 /\*

62 \* 当我们在写应用程序的时候，可能需要用到一些宏定义。

63 \*/

64

65 /\*

66 \\*

67 \* 函数声明

68 \\*

69 \*/

70 static void AppTaskCreate(void);/\* 用于创建任务 \*/

71

**72 static void Swtmr1_Callback(void\* parameter);**

**73 static void Swtmr2_Callback(void\* parameter);**

74

75 static void BSP_Init(void);/\* 用于初始化板载相关资源 \*/

76

77 /\*

78 \* @brief 主函数

79 \* @param 无

80 \* @retval 无

81 \* @note 第一步：开发板硬件初始化

82 第二步：创建APP应用任务

83 第三步：启动FreeRTOS，开始多任务调度

84 \/

85 int main(void)

86 {

87 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

88

89 /\* 开发板硬件初始化 \*/

90 BSP_Init();

91

92 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS软件定时器实验！\n");

93

94 /\* 创建AppTaskCreate任务 \*/

95 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,/\* 任务入口函数 \*/

96 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

97 (uint16_t )512, /\* 任务栈大小 \*/

98 (void\* )NULL,/\* 任务入口函数参数 \*/

99 (UBaseType_t )1, /\* 任务的优先级 \*/

100 (TaskHandle_t*)&AppTaskCreate_Handle);/\* 任务控制块指针 \*/

101 /\* 启动任务调度 \*/

102 if (pdPASS == xReturn)

103 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

104 else

105 return -1;

106

107 while (1); /\* 正常不会执行到这里 \*/

108 }

109

110

111 /\*

112 \* @ 函数名： AppTaskCreate

113 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

114 \* @ 参数：无

115 \* @ 返回值：无

116 \/

117 static void AppTaskCreate(void)

118 {

119 taskENTER_CRITICAL(); //进入临界区

120

121 /\*

122 \* 创建软件周期定时器

123 \* 函数原型

124 \* TimerHandle_t xTimerCreate(const char \* const pcTimerName,

125 const TickType_t xTimerPeriodInTicks,

126 const UBaseType_t uxAutoReload,

127 void \* const pvTimerID,

128 TimerCallbackFunction_t pxCallbackFunction )

129 \* @uxAutoReload : pdTRUE为周期模式，pdFALS为单次模式

130 \* 单次定时器，周期(1000个时钟节拍)，周期模式

131 \/

**132 Swtmr1_Handle=xTimerCreate((const char*)"AutoReloadTimer",**

**133 (TickType_t)1000,/*定时器周期 1000(tick) \*/**

**134 (UBaseType_t)pdTRUE,/\* 周期模式 \*/**

**135 (void*)1,/*为每个计时器分配一个索引的唯一ID \*/**

**136 (TimerCallbackFunction_t)Swtmr1_Callback);**

**137 if (Swtmr1_Handle != NULL) {**

**138 /\**

**139 \* xTicksToWait:如果在调用xTimerStart()时队列已满，则以tick为单位指定调用任务应保持**

**140 \* 在Blocked(阻塞)状态以等待start命令成功发送到timer命令队列的时间。**

**141 \* 如果在启动调度程序之前调用xTimerStart()，则忽略xTicksToWait。在这里设置等待时间为0**

**142 \/**

**143**

**144 xTimerStart(Swtmr1_Handle,0); //开启周期定时器**

**145 }**

146 /\*

147 \* 创建软件周期定时器

148 \* 函数原型

149 \* TimerHandle_t xTimerCreate(const char \* const pcTimerName,

150 const TickType_t xTimerPeriodInTicks,

151 const UBaseType_t uxAutoReload,

152 void \* const pvTimerID,

153 TimerCallbackFunction_t pxCallbackFunction )

154 \* @uxAutoReload : pdTRUE为周期模式，pdFALS为单次模式

155 \* 单次定时器，周期(5000个时钟节拍)，单次模式

156 \/

**157 Swtmr2_Handle=xTimerCreate((const char\* )"OneShotTimer",**

**158 (TickType_t)5000,/*定时器周期 5000(tick) \*/**

**159 (UBaseType_t )pdFALSE,/\* 单次模式 \*/**

**160 (void*)2,/*为每个计时器分配一个索引的唯一ID \*/**

**161 (TimerCallbackFunction_t)Swtmr2_Callback);**

**162 if (Swtmr2_Handle != NULL) {**

**163 /\**

**164 \* xTicksToWait:如果在调用xTimerStart()时队列已满，则以tick为单位指定调用任务应保持**

**165 \* 在Blocked(阻塞)状态以等待start命令成功发送到timer命令队列的时间。**

**166 \* 如果在启动调度程序之前调用xTimerStart()，则忽略xTicksToWait。在这里设置等待时间为0.**

**167 \/**

**168 xTimerStart(Swtmr2_Handle,0); //开启周期定时器**

**169 }**

170

171 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

172

173 taskEXIT_CRITICAL(); //退出临界区

174 }

175

176 /\*

177 \* @ 函数名： Swtmr1_Callback

178 \* @ 功能说明：软件定时器1 回调函数，打印回调函数信息&当前系统时间

179 \* 软件定时器请不要调用阻塞函数，也不要进行死循环，应快进快出

180 \* @ 参数：无

181 \* @ 返回值：无

182 \/

**183 static void Swtmr1_Callback(void\* parameter)**

**184 {**

**185 TickType_t tick_num1;**

**186**

**187 TmrCb_Count1++; /\* 每回调一次加一 \*/**

**188**

**189 tick_num1 = xTaskGetTickCount(); /\* 获取滴答定时器的计数值 \*/**

**190**

**191 LED1_TOGGLE;**

**192**

**193 printf("swtmr1_callback函数执行 %d 次\n", TmrCb_Count1);**

**194 printf("滴答定时器数值=%d\n", tick_num1);**

**195 }**

196

197 /\*

198 \* @ 函数名： Swtmr2_Callback

199 \* @ 功能说明：软件定时器2 回调函数，打印回调函数信息&当前系统时间

200 \* 软件定时器请不要调用阻塞函数，也不要进行死循环，应快进快出

201 \* @ 参数：无

202 \* @ 返回值：无

203 \/

**204 static void Swtmr2_Callback(void\* parameter)**

**205 {**

**206 TickType_t tick_num2;**

**207**

**208 TmrCb_Count2++; /\* 每回调一次加一 \*/**

**209**

**210 tick_num2 = xTaskGetTickCount(); /\* 获取滴答定时器的计数值 \*/**

**211**

**212 printf("swtmr2_callback函数执行 %d 次\n", TmrCb_Count2);**

**213 printf("滴答定时器数值=%d\n", tick_num2);**

**214 }**

215

216

217 /\*

218 \* @ 函数名： BSP_Init

219 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

220 \* @ 参数：

221 \* @ 返回值：无

222 \/

223 static void BSP_Init(void)

224 {

225 /\*

226 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

227 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

228 \* 都统一用这个优先级分组，千万不要再分组，切忌。

229 \*/

230 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

231

232 /\* LED 初始化 \*/

233 LED_GPIO_Config();

234

235 /\* 串口初始化 \*/

236 USART_Config();

237

238 /\* 按键初始化 \*/

239 Key_GPIO_Config();

240

241 }

242

243 /END OF FILE/

软件定时器实验现象
~~~~~~~~~

程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，在串口调试助手中可以看到运行
结果我们可以看到，每1000个tick时候软件定时器就会触发一次回调函数，当5000个tick到来的时候，触发软件定时器单次模式的回调函数，之后便不会再次调用了，具体见图19‑4。

|softwa005|

图19‑4软件定时器实验现象

.. |softwa002| image:: media\softwa002.png
   :width: 5.76806in
   :height: 2.35771in
.. |softwa003| image:: media\softwa003.png
   :width: 4.95455in
   :height: 3.91149in
.. |softwa004| image:: media\softwa004.png
   :width: 5.57901in
   :height: 2.97403in
.. |softwa005| image:: media\softwa005.png
   :width: 5.60736in
   :height: 3.47897in
