.. vim: syntax=rst

支持多优先级
------

在本章之前，FreeRTOS还没有支持多优先级，只支持两个任务互相切换，从本章开始，任务中我们开始加入优先级的功能。在FreeRTOS中，数字优先级越小，逻辑优先级也越小，这与隔壁的RT-Thread和μC/OS刚好相反。

如何支持多优先级
~~~~~~~~

就绪列表pxReadyTasksLists[ configMAX_PRIORITIES ]是一个数组，数组里面存的是就绪任务的TCB（准确来说是TCB里面的xStateListItem节点），数组的下标对应任务的优先级，优先级越低对应的数组下标越小。空闲任务的优先级最低，对应的是下标为0的链表。图8
‑1演示的是就绪列表中有两个任务就绪，优先级分别为1和2，其中空闲任务没有画出来，空闲任务自系统启动后会一直就绪，因为系统至少得保证有一个任务可以运行。

|multip002|

图8‑1就绪列表中有两个任务就绪

任务在创建的时候，会根据任务的优先级将任务插入到就绪列表不同的位置。相同优先级的任务插入到就绪列表里面的同一条链表中，这就是我们下一章要讲解的支持时间片。

pxCurrenTCB是一个全局的TCB指针，用于指向优先级最高的就绪任务的TCB，即当前正在运行的TCB。那么我们要想让任务支持优先级，即只要解决在任务切换（taskYIELD）的时候，让pxCurrenTCB指向最高优先级的就绪任务的TCB就可以，前面的章节我们是手动地让pxCurrenTCB在
任务1、任务2和空闲任务中轮转，现在我们要改成pxCurrenTCB在任务切换的时候指向最高优先级的就绪任务的TCB即可，那问题的关键就是：如果找到最高优先级的就绪任务的TCB。FreeRTOS提供了两套方法，一套是通用的，一套是根据特定的处理器优化过的，接下来我们重点讲解下这两个方法。

讲解查找最高优先级的就绪任务相关代码
~~~~~~~~~~~~~~~~~~

寻找最高优先级的就绪任务相关代码在task.c中定义，具体见代码清单8‑1。

代码清单8‑1查找最高优先级的就绪任务的相关代码

1 /\* 查找最高优先级的就绪任务：通用方法 \*/

2 #if ( configUSE_PORT_OPTIMISED_TASK_SELECTION == 0 )\ **(1)**

3 /\* uxTopReadyPriority 存的是就绪任务的最高优先级 \*/

4 #define taskRECORD_READY_PRIORITY( uxPriority )\\\ **(2)**

5 {\\

6 if( ( uxPriority ) > uxTopReadyPriority )\\

7 {\\

8 uxTopReadyPriority = ( uxPriority );\\

9 }\\

10 }/\* taskRECORD_READY_PRIORITY \*/

11

12 /*-----------------------------------------------------------*/

13

14 #define taskSELECT_HIGHEST_PRIORITY_TASK()\\\ **(3)**

15 {\\

16 UBaseType_t uxTopPriority = uxTopReadyPriority;\\\ **(3)-①**

17 /\* 寻找包含就绪任务的最高优先级的队列 \*/\\\ **(3)-②**

18 while( listLIST_IS_EMPTY( &( pxReadyTasksLists[ uxTopPriority ] ) ) )\\

19 {\\

20 --uxTopPriority;\\

21 }\\

22 /\* 获取优先级最高的就绪任务的TCB，然后更新到pxCurrentTCB \*/\\

23 listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB, &(pxReadyTasksLists[ uxTopPriority ]));\\\ **(3)-③**

24 /\* 更新uxTopReadyPriority \*/\\

25 uxTopReadyPriority = uxTopPriority;\\\ **(3)-④**

26 }/\* taskSELECT_HIGHEST_PRIORITY_TASK \*/

27

28 /*-----------------------------------------------------------*/

29

30 /\* 这两个宏定义只有在选择优化方法时才用，这里定义为空 \*/

31 #define taskRESET_READY_PRIORITY( uxPriority )

32 #define portRESET_READY_PRIORITY( uxPriority, uxTopReadyPriority )

33

34 /\* 查找最高优先级的就绪任务：根据处理器架构优化后的方法 \*/

35 #else/\* configUSE_PORT_OPTIMISED_TASK_SELECTION \*/**(4)**

36

37 #define taskRECORD_READY_PRIORITY( uxPriority ) \\\ **(5)**

38 portRECORD_READY_PRIORITY( uxPriority, uxTopReadyPriority )

39

40 /*-----------------------------------------------------------*/

41

42 #define taskSELECT_HIGHEST_PRIORITY_TASK()\\\ **(7)**

43 {\\

44 UBaseType_t uxTopPriority;\\

45 /\* 寻找最高优先级 \*/\\

46 portGET_HIGHEST_PRIORITY( uxTopPriority, uxTopReadyPriority );\\\ **(7)-①**

47 /\* 获取优先级最高的就绪任务的TCB，然后更新到pxCurrentTCB \*/\\

48 listGET_OWNER_OF_NEXT_ENTRY( pxCurrentTCB, &( pxReadyTasksLists[ uxTopPriority ] ) );\\\ **(7)-②**

49 }/\* taskSELECT_HIGHEST_PRIORITY_TASK() \*/

50

51 /*-----------------------------------------------------------*/

52 #if 0

53 #define taskRESET_READY_PRIORITY( uxPriority )\\\ **(注意)**

54 {\\

55 if(listCURRENT_LIST_LENGTH(&(pxReadyTasksLists[( uxPriority)]))==(UBaseType_t)0)\\

56 {\\

57 portRESET_READY_PRIORITY( ( uxPriority ), ( uxTopReadyPriority ) );\\

58 }\\

59 }

60 #else

61 #define taskRESET_READY_PRIORITY( uxPriority )\\\ **(6)**

62 {\\

63 portRESET_READY_PRIORITY((uxPriority ), (uxTopReadyPriority));\\

64 }

65 #endif

66

67 #endif/\* configUSE_PORT_OPTIMISED_TASK_SELECTION \*/

代码清单8‑1\ **(1)**\
：查找最高优先级的就绪任务有两种方法，具体由configUSE_PORT_OPTIMISED_TASK_SELECTION这个宏控制，定义为0选择通用方法，定义为1选择根据处理器优化的方法，该宏默认在portmacro.h中定义为1，即使用优化过的方法，但是通用方法我们也讲解下。

通用方法
^^^^

taskRECORD_READY_PRIORITY()
'''''''''''''''''''''''''''

代码清单8‑1\ **(2)**\
：taskRECORD_READY_PRIORITY()用于更新uxTopReadyPriority的值。uxTopReadyPriority是一个在task.c中定义的静态变量，用于表示创建的任务的最高优先级，默认初始化为0，即空闲任务的优先级，具体实现见代码清单8‑2。

代码清单8‑2uxTopReadyPriority定义

1 /\* 空闲任务优先级宏定义，在task.h中定义 \*/

2 #define tskIDLE_PRIORITY ( ( UBaseType_t ) 0U )

3

4 /\* 定义uxTopReadyPriority，在task.c中定义 \*/

5 staticvolatile UBaseType_t uxTopReadyPriority = tskIDLE_PRIORITY;

taskSELECT_HIGHEST_PRIORITY_TASK()
''''''''''''''''''''''''''''''''''

代码清单8‑1\ **(3)**\ ：taskSELECT_HIGHEST_PRIORITY_TASK()用于寻找优先级最高的就绪任务，实质就是更新uxTopReadyPriority和pxCurrentTCB的值。

代码清单8‑1\ **(3)-①**\ ：将uxTopReadyPriority的值暂存到局部变量uxTopPriority，接下来需要用到。

代码清单8‑1\ **(3)-②**\ ：从最高优先级对应的就绪列表数组下标开始寻找当前链表下是否有任务存在，如果没有，则uxTopPriority减一操作，继续寻找下一个优先级对应的链表中是否有任务存在，如果有则跳出while循环，表示找到了最高优先级的就绪任务。之所以可以采用从最高优先级往下搜索
，是因为任务的优先级与就绪列表的下标是一一对应的，优先级越高，对应的就绪列表数组的下标越大。

代码清单8‑1\ **(3)-③**\ ：获取优先级最高的就绪任务的TCB，然后更新到pxCurrentTCB。

代码清单8‑1\ **(3)-④**\ ：更新uxTopPriority的值到uxTopReadyPriority。

优化方法
^^^^

代码清单8‑1\ **(4)**\ ：优化的方法，这得益于Cortex-M内核有一个计算前导零的指令CLZ，所谓前导零就是计算一个变量（Cortex-
M内核单片机的变量为32位）从高位开始第一次出现1的位的前面的零的个数。比如：一个32位的变量uxTopReadyPriority，其位0、位24和位25均置1，其余位为0，具体见。那么使用前导零指令__CLZ
(uxTopReadyPriority)可以很快的计算出uxTopReadyPriority的前导零的个数为6。

|multip003|

图8‑2uxTopReadyPriority位展示

如果uxTopReadyPriority的每个位号对应的是任务的优先级，任务就绪时，则将对应的位置1，反之则清零。那么图8‑2就表示优先级0、优先级24和优先级25这三个任务就绪，其中优先级为25的任务优先级最高。利用前导零计算指令可以很快计算出就绪任务中的最高优先级为：( 31UL - (
uint32_t ) \__clz( ( uxReadyPriorities ) ) ) = ( 31UL - ( uint32_t ) 6 )=25。

.. _taskrecord_ready_priority-1:

taskRECORD_READY_PRIORITY()
'''''''''''''''''''''''''''

代码清单8‑1\ **(5)**\ ：taskRECORD_READY_PRIORITY()用于根据传进来的形参（通常形参就是任务的优先级）将变量uxTopReadyPriority的某个位置1。uxTopReadyPriority是一个在task.c中定义的静态变量，默认初始化为0。与通用方法中用
来表示创建的任务的最高优先级不一样，它在优化方法中担任的是一个优先级位图表的角色，即该变量的每个位对应任务的优先级，如果任务就绪，则将对应的位置1，反之清零。根据这个原理，只需要计算出uxTopReadyPriority的前导零个数就算找到了就绪任务的最高优先级。与taskRECORD_READY_
PRIORITY()作用相反的是taskRESET_READY_PRIORITY()。taskRECORD_READY_PRIORITY()与taskRESET_READY_PRIORITY()具体的实现见代码清单8‑3。

代码清单8‑3taskRECORD_READY_PRIORITY()taskRESET_READY_PRIORITY()（portmacro.h中定义）

1 #define portRECORD_READY_PRIORITY( uxPriority, uxReadyPriorities )\\

2 ( uxReadyPriorities ) \|= ( 1UL << ( uxPriority ) )

3

4 #define portRESET_READY_PRIORITY( uxPriority, uxReadyPriorities )\\

5 ( uxReadyPriorities ) &= ~( 1UL << ( uxPriority ) )

taskRESET_READY_PRIORITY()
''''''''''''''''''''''''''

代码清单8‑1\ **(6)**\ ：taskRESET_READY_PRIORITY()用于根据传进来的形参（通常形参就是任务的优先级）将变量uxTopReadyPriority的某个位清零。

代码清单8‑1\ **(注意)**\ ：实际上根据优先级调用taskRESET_READY_PRIORITY()函数复位uxTopReadyPriority变量中对应的位时，要先确保就绪列表中对应该优先级下的链表没有任务才行。但是我们当前实现的阻塞延时方案还是通过扫描就绪列表里面的TCB的延时变量x
TicksToDelay来实现的，还没有单独实现延时列表（任务延时列表将在下一个章节讲解），所以任务非就绪时暂时不能将任务从就绪列表移除，而是仅仅通过将任务优先级在变量uxTopReadyPriority中对应的位清零。在下一章我们实现任务延时列表之后，任务非就绪时，不仅会将任务优先级在变量uxTo
pReadyPriority中对应的位清零，还会降任务从就绪列表删除。

.. _taskselect_highest_priority_task-1:

taskSELECT_HIGHEST_PRIORITY_TASK()
''''''''''''''''''''''''''''''''''

代码清单8‑1\ **(7)**\ ：taskSELECT_HIGHEST_PRIORITY_TASK()用于寻找优先级最高的就绪任务，实质就是更新uxTopReadyPriority和pxCurrentTCB的值。

代码清单8‑1\ **(7)-①**\ ：根据uxTopReadyPriority的值，找到最高优先级，然后更新到uxTopPriority这个局部变量中。portGET_HIGHEST_PRIORITY()具体的宏实现见代码清单8‑4，在portmacro.h中定义。

代码清单8‑4portGET_HIGHEST_PRIORITY()宏定义

1 #define portGET_HIGHEST_PRIORITY( uxTopPriority, uxReadyPriorities )\\

2 uxTopPriority = ( 31UL - ( uint32_t ) \__clz( ( uxReadyPriorities ) ) )

代码清单8‑1\ **(7)-②**\ ：根据uxTopPriority的值，从就绪列表中找到就绪的最高优先级的任务的TCB，然后将TCB更新到pxCurrentTCB。

修改代码，支持多优先级
~~~~~~~~~~~

接下来我们在上一章的代码上，继续迭代修改，从而实现多优先级。

修改任务控制块
^^^^^^^

在任务控制块中增加与优先级相关的成员，具体见代码清单8‑5加粗部分。

代码清单8‑5修改任务控制块代码，增加优先级相关成员

1 typedefstruct tskTaskControlBlock

2 {

3 volatile StackType_t \*pxTopOfStack; /\* 栈顶 \*/

4

5 ListItem_t xStateListItem; /\* 任务节点 \*/

6

7 StackType_t \*pxStack; /\* 任务栈起始地址 \*/

8 /\* 任务名称，字符串形式 \*/

9 char pcTaskName[ configMAX_TASK_NAME_LEN ];

10

11 TickType_t xTicksToDelay;

**12 UBaseType_t uxPriority;**

13 } tskTCB;

修改xTaskCreateStatic()函数
^^^^^^^^^^^^^^^^^^^^^^^

修改任务创建xTaskCreateStatic()函数，具体见代码清单8‑6的加粗部分。

代码清单8‑6xTaskCreateStatic()函数

1 TaskHandle_t

2 xTaskCreateStatic(TaskFunction_t pxTaskCode,

3 const char \* const pcName,

4 const uint32_t ulStackDepth,

5 void \* const pvParameters,

**6 /\* 任务优先级，数值越大，优先级越高 \*/**

**7 UBaseType_t uxPriority,(1)**

8 StackType_t \* const puxStackBuffer,

9 TCB_t \* const pxTaskBuffer )

10 {

11 TCB_t \*pxNewTCB;

12 TaskHandle_t xReturn;

13

14 if ( ( pxTaskBuffer != NULL ) && ( puxStackBuffer != NULL ) )

15 {

16 pxNewTCB = ( TCB_t \* ) pxTaskBuffer;

17 pxNewTCB->pxStack = ( StackType_t \* ) puxStackBuffer;

18

19 /\* 创建新的任务 \*/**(2)**

**20 prvInitialiseNewTask( pxTaskCode,**

**21 pcName,**

**22 ulStackDepth,**

**23 pvParameters,**

**24 uxPriority,**

**25 &xReturn,**

**26 pxNewTCB);**

27

28 /\* 将任务添加到就绪列表 \*/**(3)**

**29 prvAddNewTaskToReadyList( pxNewTCB );**

30

31 }

32 else

33 {

34 xReturn = NULL;

35 }

36

37 return xReturn;

38 }

代码清单8‑6\ **(1)**\ ：增加优先级形参，数值越大，优先级越高。

prvInitialiseNewTask()函数
''''''''''''''''''''''''

代码清单8‑6\ **(2)**\ ：修改prvInitialiseNewTask()函数，增加优先级形参和优先级初始化相关代码，具体修改见代码清单8‑7的加粗部分。

代码清单8‑7prvInitialiseNewTask()函数

1 static void prvInitialiseNewTask(TaskFunction_t pxTaskCode,

2 const char \* const pcName,

3 const uint32_t ulStackDepth,

4 void \* const pvParameters,

**5 /\* 任务优先级，数值越大，优先级越高 \*/**

**6 UBaseType_t uxPriority,**

7 TaskHandle_t \* const pxCreatedTask,

8 TCB_t \*pxNewTCB )

9

10 {

11 StackType_t \*pxTopOfStack;

12 UBaseType_t x;

13

14 /\* 获取栈顶地址 \*/

15 pxTopOfStack = pxNewTCB->pxStack + ( ulStackDepth - ( uint32_t ) 1 );

16 /\* 向下做8字节对齐 \*/

17 pxTopOfStack = ( StackType_t \* ) ( ( ( uint32_t ) pxTopOfStack ) & ( ~( ( uint32_t ) 0x0007 ) ) );

18

19 /\* 将任务的名字存储在TCB中 \*/

20 for ( x = ( UBaseType_t ) 0; x < ( UBaseType_t ) configMAX_TASK_NAME_LEN; x++ )

21 {

22 pxNewTCB->pcTaskName[ x ] = pcName[ x ];

23

24 if ( pcName[ x ] == 0x00 )

25 {

26 break;

27 }

28 }

29 /\* 任务名字的长度不能超过configMAX_TASK_NAME_LEN \*/

30 pxNewTCB->pcTaskName[ configMAX_TASK_NAME_LEN - 1 ] = '\0';

31

32 /\* 初始化TCB中的xStateListItem节点 \*/

33 vListInitialiseItem( &( pxNewTCB->xStateListItem ) );

34 /\* 设置xStateListItem节点的拥有者 \*/

35 listSET_LIST_ITEM_OWNER( &( pxNewTCB->xStateListItem ), pxNewTCB );

36

**37 /\* 初始化优先级 \*/**

**38 if ( uxPriority >= ( UBaseType_t ) configMAX_PRIORITIES )**

**39 {**

**40 uxPriority = ( UBaseType_t ) configMAX_PRIORITIES - ( UBaseType_t ) 1U;**

**41 }**

**42 pxNewTCB->uxPriority = uxPriority;**

43

44 /\* 初始化任务栈 \*/

45 pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxTaskCode, pvParameters );

46

47 /\* 让任务句柄指向任务控制块 \*/

48 if ( ( void \* ) pxCreatedTask != NULL )

49 {

50 \*pxCreatedTask = ( TaskHandle_t ) pxNewTCB;

51 }

52 }

prvAddNewTaskToReadyList()函数
''''''''''''''''''''''''''''

代码清单8‑6\ **(3)**\ ：新增将任务添加到就绪列表的函数prvAddNewTaskToReadyList()，该函数在task.c中实现，具体见代码清单8‑8。

代码清单8‑8prvAddNewTaskToReadyList()函数

1 static void prvAddNewTaskToReadyList( TCB_t \*pxNewTCB )

2 {

3 /\* 进入临界段 \*/

4 taskENTER_CRITICAL();

5 {

6 /\* 全局任务计时器加一操作 \*/

7 uxCurrentNumberOfTasks++;\ **(1)**

8

9 /\* 如果pxCurrentTCB为空，则将pxCurrentTCB指向新创建的任务 \*/

10 if ( pxCurrentTCB == NULL )\ **(2)**

11 {

12 pxCurrentTCB = pxNewTCB;

13

14 /\* 如果是第一次创建任务，则需要初始化任务相关的列表 \*/

15 if ( uxCurrentNumberOfTasks == ( UBaseType_t ) 1 )\ **(3)**

16 {

17 /\* 初始化任务相关的列表 \*/

18 prvInitialiseTaskLists();

19 }

20 }

21 else/\* 如果pxCurrentTCB不为空，\ **(4)**

22 则根据任务的优先级将pxCurrentTCB指向最高优先级任务的TCB \*/

23 {

24 if ( pxCurrentTCB->uxPriority <= pxNewTCB->uxPriority )

25 {

26 pxCurrentTCB = pxNewTCB;

27 }

28 }

29

30 /\* 将任务添加到就绪列表 \*/

31 prvAddTaskToReadyList( pxNewTCB );\ **(5)**

32

33 }

34 /\* 退出临界段 \*/

35 taskEXIT_CRITICAL();

36 }

代码清单8‑8\ **(1)**\ ：全局任务计时器uxCurrentNumberOfTasks加一操作。uxCurrentNumberOfTasks是一个在task.c中定义的静态变量，默认初始化为0

代码清单8‑8\ **(2)**\ ：如果pxCurrentTCB为空，则将pxCurrentTCB指向新创建的任务。pxCurrentTCB是一个在task.c定义的全局指针，用于指向当前正在运行或者即将要运行的任务的任务控制块，默认初始化为NULL。

代码清单8‑8\ **(3)**\ ：如果是第一次创建任务，则需要调用函数prvInitialiseTaskLists()初始化任务相关的列表，目前只有就绪列表需要初始化，该函数在task.c中定义，具体实现见代码清单8‑9。

prvInitialiseTaskLists()函数


代码清单8‑9prvInitialiseTaskLists()函数

1 /\* 初始化任务相关的列表 \*/

2 void prvInitialiseTaskLists( void )

3 {

4 UBaseType_t uxPriority;

5

6 for ( uxPriority = ( UBaseType_t ) 0U; uxPriority < ( UBaseType_t ) configMAX_PRIORITIES; uxPriority++ )

7 {

8 vListInitialise( &( pxReadyTasksLists[ uxPriority ] ) );

9 }

10 }

代码清单8‑8\ **(4)**\ ：如果pxCurrentTCB不为空，表示当前已经有任务存在，则根据任务的优先级将pxCurrentTCB指向最高优先级任务的TCB。在创建任务时，始终让pxCurrentTCB指向最高优先级任务的TCB。

代码清单8‑8\ **(5)**\ ：将任务添加到就绪列表。prvAddTaskToReadyList()是一个带参宏，在task.c中定义，具体实现见代码清单8‑10。

prvAddTaskToReadyList()函数


代码清单8‑10prvAddTaskToReadyList()函数

1 /\* 将任务添加到就绪列表 \*/

2 #define prvAddTaskToReadyList( pxTCB )\\

3 taskRECORD_READY_PRIORITY( ( pxTCB )->uxPriority );\\\ **(1)**

4 vListInsertEnd( &( pxReadyTasksLists[ ( pxTCB )->uxPriority ] ),\\\ **(2)**

5 &( ( pxTCB )->xStateListItem ) );

代码清单8‑10\ **(1)**\ ：根据优先级将优先级位图表uxTopReadyPriority中对应的位置位。

代码清单8‑10\ **(2)**\ ：根据优先级将任务插入到就绪列表pxReadyTasksLists[]。

修改vTaskStartScheduler()函数
^^^^^^^^^^^^^^^^^^^^^^^^^

修改开启任务调度函数vTaskStartScheduler()，具体见代码清单8‑11的加粗部分。

代码清单8‑11vTaskStartScheduler()函数

1 void vTaskStartScheduler( void )

2 {

3 /*======================创建空闲任务start==========================*/

4 TCB_t \*pxIdleTaskTCBBuffer = NULL;

5 StackType_t \*pxIdleTaskStackBuffer = NULL;

6 uint32_t ulIdleTaskStackSize;

7

8 /\* 获取空闲任务的内存：任务栈和任务TCB \*/

9 vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer,

10 &pxIdleTaskStackBuffer,

11 &ulIdleTaskStackSize );

12

13 xIdleTaskHandle =

14 xTaskCreateStatic( (TaskFunction_t)prvIdleTask,

15 (char \*)"IDLE",

16 (uint32_t)ulIdleTaskStackSize ,

17 (void \*) NULL,

**18 /\* 任务优先级，数值越大，优先级越高 \*/**

**19 (UBaseType_t) tskIDLE_PRIORITY,(1)**

20 (StackType_t \*)pxIdleTaskStackBuffer,

21 (TCB_t \*)pxIdleTaskTCBBuffer );

22 /\* 将任务添加到就绪列表 \*/**(2)**

23 /\* vListInsertEnd( &( pxReadyTasksLists[0] ),

24 &( ((TCB_t \*)pxIdleTaskTCBBuffer)->xStateListItem ) ); \*/

25 /*===================创建空闲任务end=========================*/

26

27 /\* 手动指定第一个运行的任务 \*/**(3)**

28 //pxCurrentTCB = &Task1TCB;

29

30 /\* 启动调度器 \*/

31 if ( xPortStartScheduler() != pdFALSE )

32 {

33 /\* 调度器启动成功，则不会返回，即不会来到这里 \*/

34 }

35 }

代码清单8‑11\ **(1)**\ ：创建空闲任务时，优先级配置为tskIDLE_PRIORITY，该宏在task.h中定义，默认为0，表示空闲任务的优先级为最低。

代码清单8‑11\ **(2)**\ ：刚刚我们已经修改了创建任务函数xTaskCreateStatic()，在创建任务时，就已经将任务添加到了就绪列表，这里将注释掉。

代码清单8‑11\ **(3)**\ ：在刚刚修改的创建任务函数xTaskCreateStatic()中，增加了将任务添加到就绪列表的函数prvAddNewTaskToReadyList()，这里将注释掉。

修改vTaskDelay()函数
^^^^^^^^^^^^^^^^

vTaskDelay()函数修改内容是添加了将任务从就绪列表移除的操作，具体实现见代码清单8‑12加粗部分。

代码清单8‑12vTaskDelay()函数

1 void vTaskDelay( const TickType_t xTicksToDelay )

2 {

3 TCB_t \*pxTCB = NULL;

4

5 /\* 获取当前任务的TCB \*/

6 pxTCB = pxCurrentTCB;

7

8 /\* 设置延时时间 \*/

9 pxTCB->xTicksToDelay = xTicksToDelay;

10

**11 /\* 将任务从就绪列表移除 \*/**

**12 //uxListRemove( &( pxTCB->xStateListItem ) );(注意)**

**13 taskRESET_READY_PRIORITY( pxTCB->uxPriority );**

14

15 /\* 任务切换 \*/

16 taskYIELD();

17 }

代码清单8‑12\ **(注意)**\ ：将任务从就绪列表移除本应该完成两个操作：1个是将任务从就绪列表移除，由函数uxListRemove()来实现；另一个是根据优先级将优先级位图表uxTopReadyPriority中对应的位清零，由函数taskRESET_READY_PRIORITY()来实现
。但是鉴于我们目前的时基更新函数xTaskIncrementTick还是需要通过扫描就绪列表的任务来判断任务的延时时间是否到期，所以不能将任务从就绪列表移除。当我们在接下来的“任务延时列表的实现”章节中，会专门添加一个延时列表，到时延时的时候除了根据优先级将优先级位图表uxTopReadyPrior
ity中对应的位清零外，还需要将任务从就绪列表移除。

修改vTaskSwitchContext()函数
^^^^^^^^^^^^^^^^^^^^^^^^

在新的任务切换函数vTaskSwitchContext()中，不再是手动的让pxCurrentTCB指针在任务1、任务2和空闲任务中切换，而是直接调用函数taskSELECT_HIGHEST_PRIORITY_TASK()寻找到优先级最高的就绪任务的TCB，然后更新到pxCurrentTCB，具体实
现见代码清单8‑13的加粗部分。

代码清单8‑13vTaskSwitchContext()函数

1 #if 1

2 /\* 任务切换，即寻找优先级最高的就绪任务 \*/

**3 void vTaskSwitchContext( void )**

**4 {**

**5 /\* 获取优先级最高的就绪任务的TCB，然后更新到pxCurrentTCB \*/**

**6 taskSELECT_HIGHEST_PRIORITY_TASK();**

**7 }**

8 #else

9 void vTaskSwitchContext( void )

10 {

11 /\* 如果当前任务是空闲任务，那么就去尝试执行任务1或者任务2，

12 看看他们的延时时间是否结束，如果任务的延时时间均没有到期，

13 那就返回继续执行空闲任务 \*/

14 if ( pxCurrentTCB == &IdleTaskTCB )

15 {

16 if (Task1TCB.xTicksToDelay == 0)

17 {

18 pxCurrentTCB =&Task1TCB;

19 }

20 else if (Task2TCB.xTicksToDelay == 0)

21 {

22 pxCurrentTCB =&Task2TCB;

23 }

24 else

25 {

26 return; /\* 任务延时均没有到期则返回，继续执行空闲任务 \*/

27 }

28 }

29 else

30 {

31 /*如果当前任务是任务1或者任务2的话，

32 检查下另外一个任务,如果另外的任务不在延时中，

33 就切换到该任务。否则，判断下当前任务是否应该进入延时状态，

34 如果是的话，就切换到空闲任务。否则就不进行任何切换 \*/

35 if (pxCurrentTCB == &Task1TCB)

36 {

37 if (Task2TCB.xTicksToDelay == 0)

38 {

39 pxCurrentTCB =&Task2TCB;

40 }

41 else if (pxCurrentTCB->xTicksToDelay != 0)

42 {

43 pxCurrentTCB = &IdleTaskTCB;

44 }

45 else

46 {

47 return; /\* 返回，不进行切换，因为两个任务都处于延时中 \*/

48 }

49 }

50 else if (pxCurrentTCB == &Task2TCB)

51 {

52 if (Task1TCB.xTicksToDelay == 0)

53 {

54 pxCurrentTCB =&Task1TCB;

55 }

56 else if (pxCurrentTCB->xTicksToDelay != 0)

57 {

58 pxCurrentTCB = &IdleTaskTCB;

59 }

60 else

61 {

62 return; /\* 返回，不进行切换，因为两个任务都处于延时中 \*/

63 }

64 }

65 }

66 }

67

68 #endif

修改xTaskIncrementTick()函数
^^^^^^^^^^^^^^^^^^^^^^^^

修改xTaskIncrementTick()函数，即在原来的基础上增加：当任务延时时间到，将任务就绪的代码，具体见代码清单8‑14的加粗部分。

代码清单8‑14xTaskIncrementTick()函数

1 void xTaskIncrementTick( void )

2 {

3 TCB_t \*pxTCB = NULL;

4 BaseType_t i = 0;

5

6 const TickType_t xConstTickCount = xTickCount + 1;

7 xTickCount = xConstTickCount;

8

9

10 /\* 扫描就绪列表中所有任务的remaining_tick，如果不为0，则减1 \*/

11 for (i=0; i<configMAX_PRIORITIES; i++)

12 {

13 pxTCB = ( TCB_t \* ) listGET_OWNER_OF_HEAD_ENTRY( ( &pxReadyTasksLists[i] ) );

14 if (pxTCB->xTicksToDelay > 0)

15 {

16 pxTCB->xTicksToDelay --;

17

**18 /\* 延时时间到，将任务就绪 \*/(增加)**

**19 if ( pxTCB->xTicksToDelay ==0 )**

**20 {**

**21 taskRECORD_READY_PRIORITY( pxTCB->uxPriority );**

**22 }**

23 }

24 }

25

26 /\* 任务切换 \*/

27 portYIELD();

28 }

代码清单8‑14\ **(增加)**\ ：延时时间到，将任务就绪。即根据优先级将优先级位图表uxTopReadyPriority中对应的位置位。在刚刚修改的上下文切换函数vTaskSwitchContext()中，就是通过优先级位图表uxTopReadyPriority来寻找就绪任务的最高优先级的。

main函数
~~~~~~

本章main函数与上一章基本一致，修改不大，具体修改见代码清单8‑15的加粗部分。

代码清单8‑15main函数

1 /\*

2 \\*

3 \* 包含的头文件

4 \\*

5 \*/

6 #include"FreeRTOS.h"

7 #include"task.h"

8

9 /\*

10 \\*

11 \* 全局变量

12 \\*

13 \*/

14 portCHAR flag1;

15 portCHAR flag2;

16

17

18 extern List_t pxReadyTasksLists[ configMAX_PRIORITIES ];

19

20

21 /\*

22 \\*

23 \* 任务控制块& STACK

24 \\*

25 \*/

26 TaskHandle_t Task1_Handle;

27 #define TASK1_STACK_SIZE 128

28 StackType_t Task1Stack[TASK1_STACK_SIZE];

29 TCB_t Task1TCB;

30

31 TaskHandle_t Task2_Handle;

32 #define TASK2_STACK_SIZE 128

33 StackType_t Task2Stack[TASK2_STACK_SIZE];

34 TCB_t Task2TCB;

35

36

37 /\*

38 \\*

39 \* 函数声明

40 \\*

41 \*/

42 void delay (uint32_t count);

43 void Task1_Entry( void \*p_arg );

44 void Task2_Entry( void \*p_arg );

45

46

47 /\*

48 \\*

49 \* main函数

50 \\*

51 \*/

52 int main(void)

53 {

54 /\* 硬件初始化 \*/

55 /\* 将硬件相关的初始化放在这里，如果是软件仿真则没有相关初始化代码 \*/

56

57

58 /\* 创建任务 \*/

59 Task1_Handle =

60 xTaskCreateStatic( (TaskFunction_t)Task1_Entry,

61 (char \*)"Task1",

62 (uint32_t)TASK1_STACK_SIZE ,

63 (void \*) NULL,

**64 /\* 任务优先级，数值越大，优先级越高 \*/(1)**

**65 (UBaseType_t) 1,**

66 (StackType_t \*)Task1Stack,

67 (TCB_t \*)&Task1TCB );

**68 /\* 将任务添加到就绪列表 \*/(2)**

**69 /\* vListInsertEnd( &( pxReadyTasksLists[1] ),**

**70 &( ((TCB_t \*)(&Task1TCB))->xStateListItem ) ); \*/**

71

72 Task2_Handle =

73 xTaskCreateStatic( (TaskFunction_t)Task2_Entry,

74 (char \*)"Task2",

75 (uint32_t)TASK2_STACK_SIZE ,

76 (void \*) NULL,

**77 /\* 任务优先级，数值越大，优先级越高 \*/(3)**

**78 (UBaseType_t) 2,**

79 (StackType_t \*)Task2Stack,

80 (TCB_t \*)&Task2TCB );

**81 /\* 将任务添加到就绪列表 \*/(4)**

**82 /\* vListInsertEnd( &( pxReadyTasksLists[2] ),**

**83 &( ((TCB_t \*)(&Task2TCB))->xStateListItem ) ); \*/**

84

85 /\* 启动调度器，开始多任务调度，启动成功则不返回 \*/

86 vTaskStartScheduler();

87

88 for (;;)

89 {

90 /\* 系统启动成功不会到达这里 \*/

91 }

92 }

93

94 /\*

95 \\*

96 \* 函数实现

97 \\*

98 \*/

99 /\* 软件延时 \*/

100 void delay (uint32_t count)

101 {

102 for (; count!=0; count--);

103 }

104 /\* 任务1 \*/

105 void Task1_Entry( void \*p_arg )

106 {

107 for ( ;; )

108 {

109 flag1 = 1;

110 vTaskDelay( 2 );

111 flag1 = 0;

112 vTaskDelay( 2 );

113 }

114 }

115

116 /\* 任务2 \*/

117 void Task2_Entry( void \*p_arg )

118 {

119 for ( ;; )

120 {

121 flag2 = 1;

122 vTaskDelay( 2 );

123 flag2 = 0;

124 vTaskDelay( 2 );

125 }

126 }

127

128

129 /\* 获取空闲任务的内存 \*/

130 StackType_t IdleTaskStack[configMINIMAL_STACK_SIZE];

131 TCB_t IdleTaskTCB;

132 void vApplicationGetIdleTaskMemory( TCB_t \**ppxIdleTaskTCBBuffer,

133 StackType_t \**ppxIdleTaskStackBuffer,

134 uint32_t \*pulIdleTaskStackSize )

135 {

136 \*ppxIdleTaskTCBBuffer=&IdleTaskTCB;

137 \*ppxIdleTaskStackBuffer=IdleTaskStack;

138 \*pulIdleTaskStackSize=configMINIMAL_STACK_SIZE;

139 }

代码清单8‑15\ **(1)和(3)**\ ：设置任务的优先级，数字优先级越高，逻辑优先级越高。

代码清单8‑15\ **(2)和(4)**\ ：这部分代码删除，因为在任务创建函数xTaskCreateStatic()中，已经调用函数prvAddNewTaskToReadyList()将任务插入到了就绪列表。。

实验现象
~~~~

进入软件调试，全速运行程序，从逻辑分析仪中可以看到两个任务的波形是完全同步，就好像CPU在同时干两件事情，具体仿真的波形图见图8‑3和图8‑4。

|multip004|

图8‑3实验现象1

|multip005|

图8‑4实验现象2

从图7‑1和图7‑2可以看出，flag1和flag2的高电平的时间为(0.1802-0.1602)s，刚好等于阻塞延时的20ms，所以实验现象跟代码要实现的功能是一致的。。

.. |multip002| image:: media\multip002.png
   :width: 4.33117in
   :height: 3.66696in
.. |multip003| image:: media\multip003.png
   :width: 5.76806in
   :height: 0.54409in
.. |multip004| image:: media\multip004.png
   :width: 4.53472in
   :height: 2.02441in
.. |multip005| image:: media\multip005.png
   :width: 4.48611in
   :height: 2.32731in
