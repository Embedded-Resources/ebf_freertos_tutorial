.. vim: syntax=rst

空闲任务与阻塞延时的实现
------------

在上一章节中，任务体内的延时使用的是软件延时，即还是让CPU空等来达到延时的效果。使用RTOS的很大优势就是榨干CPU的性能，永远不能让它闲着，任务如果需要延时也就不能再让CPU空等来实现延时的效果。RTOS中的延时叫阻塞延时，即任务需要延时的时候，任务会放弃CPU的使用权，CPU可以去干其他的事情
，当任务延时时间到，重新获取CPU使用权，任务继续运行，这样就充分地利用了CPU的资源，而不是干等着。

当任务需要延时，进入阻塞状态，那CPU又去干什么事情了？如果没有其他任务可以运行，RTOS都会为CPU创建一个空闲任务，这个时候CPU就运行空闲任务。在FreeRTOS中，空闲任务是系统在【启动调度器】的时候创建的优先级最低的任务，空闲任务主体主要是做一些系统内存的清理工作。但是为了简单起见，我们本
章实现的空闲任务只是对一个全局变量进行计数。鉴于空闲任务的这种特性，在实际应用中，当系统进入空闲任务的时候，可在空闲任务中让单片机进入休眠或者低功耗等操作。

实现空闲任务
~~~~~~

目前我们在创建任务时使用的栈和TCB都使用的是静态的内存，即需要预先定义好内存，空闲任务也不例外。有关空闲任务的栈和TCB需要用到的内存空间均在main.c中定义。

定义空闲任务的栈
^^^^^^^^

空闲任务的栈在我们在main.c中定义，具体见代码清单7‑1。

代码清单7‑1定义空闲任务的栈

1 /\* 定义空闲任务的栈 \*/

2 #define configMINIMAL_STACK_SIZE ( ( unsigned short ) 128 )\ **(2)**

3 StackType_t IdleTaskStack[configMINIMAL_STACK_SIZE];\ **(1)**

代码清单7‑1\ **(1)**\ ：空闲任务的栈是一个定义好的数组，大小由FreeRTOSConfig.h中定义的宏configMINIMAL_STACK_SIZE控制，默认为128，单位为字，即512个字节。

定义空闲任务的任务控制块
^^^^^^^^^^^^

任务控制块是每一个任务必须的，空闲任务的的任务控制块我们在main.c中定义，是一个全局变量，具体见代码清单7‑2。

代码清单7‑2定义空闲任务的任务控制块

1 /\* 定义空闲任务的任务控制块 \*/

2 TCB_t IdleTaskTCB;

创建空闲任务
^^^^^^

当定义好空闲任务的栈，任务控制块后，就可以创建空闲任务。空闲任务在调度器启动函数vTaskStartScheduler()中创建，具体实现见代码清单7‑3的加粗部分。

代码清单7‑3创建空闲任务

1 extern TCB_t IdleTaskTCB;

2 void vApplicationGetIdleTaskMemory( TCB_t \**ppxIdleTaskTCBBuffer,

3 StackType_t \**ppxIdleTaskStackBuffer,

4 uint32_t \*pulIdleTaskStackSize );

5 void vTaskStartScheduler( void )

6 {

**7 /*=======================创建空闲任务start=======================*/**

**8 TCB_t \*pxIdleTaskTCBBuffer = NULL; /\* 用于指向空闲任务控制块 \*/**

**9 StackType_t \*pxIdleTaskStackBuffer = NULL; /\* 用于空闲任务栈起始地址 \*/**

**10 uint32_t ulIdleTaskStackSize;**

**11**

**12 /\* 获取空闲任务的内存：任务栈和任务TCB \*/(1)**

**13 vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer,**

**14 &pxIdleTaskStackBuffer,**

**15 &ulIdleTaskStackSize );**

**16 /\* 创建空闲任务 \*/ (2)**

**17 xIdleTaskHandle =**

**18 xTaskCreateStatic( (TaskFunction_t)prvIdleTask, /\* 任务入口 \*/**

**19 (char \*)"IDLE", /\* 任务名称，字符串形式 \*/**

**20 (uint32_t)ulIdleTaskStackSize , /\* 任务栈大小，单位为字 \*/**

**21 (void \*) NULL, /\* 任务形参 \*/**

**22 (StackType_t \*)pxIdleTaskStackBuffer, /\* 任务栈起始地址 \*/**

**23 (TCB_t \*)pxIdleTaskTCBBuffer ); /\* 任务控制块 \*/**

**24 /\* 将任务添加到就绪列表 \*/(3)**

**25 vListInsertEnd( &( pxReadyTasksLists[0] ),**

**26 &( ((TCB_t \*)pxIdleTaskTCBBuffer)->xStateListItem ) );**

**27 /*==========================创建空闲任务end=====================*/**

28

29 /\* 手动指定第一个运行的任务 \*/

30 pxCurrentTCB = &Task1TCB;

31

32 /\* 启动调度器 \*/

33 if ( xPortStartScheduler() != pdFALSE )

34 {

35 /\* 调度器启动成功，则不会返回，即不会来到这里 \*/

36 }

37 }

代码清单7‑3\ **(1)**\ ：获取空闲任务的内存，即将pxIdleTaskTCBBuffer和pxIdleTaskStackBuffer这两个接下来要作为形参传到xTaskCreateStatic()函数的指针分别指向空闲任务的TCB和栈的起始地址，这个操作由函数vApplicationGe
tIdleTaskMemory()来实现，该函数需要用户自定义，目前我们在main.c中实现，具体见代码清单7‑4。

代码清单7‑4vApplicationGetIdleTaskMemory()函数

1 void vApplicationGetIdleTaskMemory( TCB_t \**ppxIdleTaskTCBBuffer,

2 StackType_t \**ppxIdleTaskStackBuffer,

3 uint32_t \*pulIdleTaskStackSize )

4 {

5 \*ppxIdleTaskTCBBuffer=&IdleTaskTCB;

6 \*ppxIdleTaskStackBuffer=IdleTaskStack;

7 \*pulIdleTaskStackSize=configMINIMAL_STACK_SIZE;

8 }

代码清单7‑3\ **(2)**\ ：调用xTaskCreateStatic()函数创建空闲任务。

代码清单7‑3\ **(3)**\ ：将空闲任务插入到就绪列表的开头。在下一章我们会支持优先级，空闲任务默认的优先级是最低的，即排在就绪列表的开头。

实现阻塞延时
~~~~~~

vTaskDelay()函数
^^^^^^^^^^^^^^

阻塞延时的阻塞是指任务调用该延时函数后，任务会被剥离CPU使用权，然后进入阻塞状态，直到延时结束，任务重新获取CPU使用权才可以继续运行。在任务阻塞的这段时间，CPU可以去执行其他的任务，如果其他的任务也在延时状态，那么CPU就将运行空闲任务。阻塞延时函数在task.c中定义，具体代码实现见代码清单
7‑5。

代码清单7‑5vTaskDelay()函数

1 void vTaskDelay( const TickType_t xTicksToDelay )

2 {

3 TCB_t \*pxTCB = NULL;

4

5 /\* 获取当前任务的TCB \*/

6 pxTCB = pxCurrentTCB;\ **(1)**

7

8 /\* 设置延时时间 \*/

9 pxTCB->xTicksToDelay = xTicksToDelay;\ **(2)**

10

11 /\* 任务切换 \*/

12 taskYIELD();\ **(3)**

13 }

代码清单7‑5\ **(1)**\ ：获取当前任务的任务控制块。pxCurrentTCB是一个在task.c定义的全局指针，用于指向当前正在运行或者即将要运行的任务的任务控制块。

代码清单7‑5\ **(2)**\ ：xTicksToDelay是任务控制块的一个成员，用于记录任务需要延时的时间，单位为SysTick的中断周期。比如我们本书当中SysTick的中断周期为10ms，调用vTaskDelay( 2
)则完成2*10ms的延时。xTicksToDelay定义具体见代码清单7‑6的加粗部分。

代码清单7‑6xTicksToDelay定义

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

**11 TickType_t xTicksToDelay; /\* 用于延时 \*/**

12 } tskTCB;

修改vTaskSwitchContext()函数
^^^^^^^^^^^^^^^^^^^^^^^^

代码清单7‑5\ **(3)**\ ：任务切换。调用tashYIELD()会产生PendSV中断，在PendSV中断服务函数中会调用上下文切换函数vTaskSwitchContext()，该函数的作用是寻找最高优先级的就绪任务，然后更新pxCurrentTCB。上一章我们只有两个任务，则pxCurr
entTCB不是指向任务1就是指向任务2，本章节开始我们多增加了一个空闲任务，则需要让pxCurrentTCB在这三个任务中切换，算法需要改变，具体实现见代码清单7‑7的加粗部分。

代码清单7‑7vTaskSwitchContext()函数

1 #if 0

2 void vTaskSwitchContext( void )

3 {/\* 两个任务轮流切换 \*/

4 if ( pxCurrentTCB == &Task1TCB )

5 {

6 pxCurrentTCB = &Task2TCB;

7 }

8 else

9 {

10 pxCurrentTCB = &Task1TCB;

11 }

12 }

13 #else

14

**15 void vTaskSwitchContext( void )**

**16 {**

**17 /\* 如果当前任务是空闲任务，那么就去尝试执行任务1或者任务2，**

**18 看看他们的延时时间是否结束，如果任务的延时时间均没有到期，**

**19 那就返回继续执行空闲任务 \*/**

**20 if ( pxCurrentTCB == &IdleTaskTCB )(1)**

**21 {**

**22 if (Task1TCB.xTicksToDelay == 0)**

**23 {**

**24 pxCurrentTCB =&Task1TCB;**

**25 }**

**26 else if (Task2TCB.xTicksToDelay == 0)**

**27 {**

**28 pxCurrentTCB =&Task2TCB;**

**29 }**

**30 else**

**31 {**

**32 return; /\* 任务延时均没有到期则返回，继续执行空闲任务 \*/**

**33 }**

**34 }**

**35 else/\* 当前任务不是空闲任务则会执行到这里 \*/(2)**

**36 {**

**37 /*如果当前任务是任务1或者任务2的话，检查下另外一个任务,**

**38 如果另外的任务不在延时中，就切换到该任务**

**39 否则，判断下当前任务是否应该进入延时状态，**

**40 如果是的话，就切换到空闲任务。否则就不进行任何切换 \*/**

**41 if (pxCurrentTCB == &Task1TCB)**

**42 {**

**43 if (Task2TCB.xTicksToDelay == 0)**

**44 {**

**45 pxCurrentTCB =&Task2TCB;**

**46 }**

**47 else if (pxCurrentTCB->xTicksToDelay != 0)**

**48 {**

**49 pxCurrentTCB = &IdleTaskTCB;**

**50 }**

**51 else**

**52 {**

**53 return; /\* 返回，不进行切换，因为两个任务都处于延时中 \*/**

**54 }**

**55 }**

**56 else if (pxCurrentTCB == &Task2TCB)**

**57 {**

**58 if (Task1TCB.xTicksToDelay == 0)**

**59 {**

**60 pxCurrentTCB =&Task1TCB;**

**61 }**

**62 else if (pxCurrentTCB->xTicksToDelay != 0)**

**63 {**

**64 pxCurrentTCB = &IdleTaskTCB;**

**65 }**

**66 else**

**67 {**

**68 return; /\* 返回，不进行切换，因为两个任务都处于延时中 \*/**

**69 }**

**70 }**

**71 }**

**72 }**

73

74 #endif

代码清单7‑7\ **(1)**\ ：如果当前任务是空闲任务，那么就去尝试执行任务1或者任务2，看看他们的延时时间是否结束，如果任务的延时时间均没有到期，那就返回继续执行空闲任务。

代码清单7‑7\ **(2)**\ ：如果当前任务是任务1或者任务2的话，检查下另外一个任务，如果另外的任务不在延时中，就切换到该任务。否则，判断下当前任务是否应该进入延时状态，如果是的话，就切换到空闲任务，否则就不进行任何切换。

SysTick中断服务函数
~~~~~~~~~~~~~

在任务上下文切换函数vTaskSwitchContext()中，会判断每个任务的任务控制块中的延时成员xTicksToDelay的值是否为0，如果为0就要将对应的任务就绪，如果不为0就继续延时。如果一个任务要延时，一开始xTicksToDelay肯定不为0，当xTicksToDelay变为0的时候表
示延时结束，那么xTicksToDelay是以什么周期在递减？在哪里递减？在FreeRTOS中，这个周期由SysTick中断提供，操作系统里面的最小的时间单位就是SysTick的中断周期，我们称之为一个tick，SysTick中断服务函数在port.c.c中实现，具体见\
**错误！未找到引用源。**\ 。

代码清单7‑8SysTick中断服务函数

1 void xPortSysTickHandler( void )

2 {

3 /\* 关中断 \*/

4 vPortRaiseBASEPRI();\ **(1)**

5

6 /\* 更新系统时基 \*/

7 xTaskIncrementTick();\ **(2)**

8

9 /\* 开中断 \*/

10 vPortClearBASEPRIFromISR();\ **(3)**

11 }

代码清单7‑8\ **(1)**\ ：进入临界段，关中断。

xTaskIncrementTick()函数
^^^^^^^^^^^^^^^^^^^^^^

代码清单7‑8\ **(2)**\ ：更新系统时基，该函数在task.c中定义，具体见代码清单7‑9。

代码清单7‑9xTaskIncrementTick()函数

1 void xTaskIncrementTick( void )

2 {

3 TCB_t \*pxTCB = NULL;

4 BaseType_t i = 0;

5

6 /\* 更新系统时基计数器xTickCount，xTickCount是一个在port.c中定义的全局变量 \*/**(1)**

7 const TickType_t xConstTickCount = xTickCount + 1;

8 xTickCount = xConstTickCount;

9

10

11 /\* 扫描就绪列表中所有任务的xTicksToDelay，如果不为0，则减1 \*/**(2)**

12 for (i=0; i<configMAX_PRIORITIES; i++)

13 {

14 pxTCB = ( TCB_t \* ) listGET_OWNER_OF_HEAD_ENTRY( ( &pxReadyTasksLists[i] ) );

15 if (pxTCB->xTicksToDelay > 0)

16 {

17 pxTCB->xTicksToDelay --;

18 }

19 }

20

21 /\* 任务切换 \*/**(3)**

22 portYIELD();

23 }

代码清单7‑9\ **(1)**\ ：更新系统时基计数器xTickCount，加一操作。xTickCount是一个在port.c中定义的全局变量，在函数vTaskStartScheduler()中调用xPortStartScheduler()函数前初始化。

代码清单7‑9\ **(2)**\ ：扫描就绪列表中所有任务的xTicksToDelay，如果不为0，则减1。

代码清单7‑9\ **(3)**\ ：执行一次任务切换。

代码清单7‑8\ **(3)**\ ：退出临界段，开中断。

SysTick初始化函数
~~~~~~~~~~~~

SysTick的中断服务函数要想被顺利执行，则SysTick必须先初始化。SysTick初始化函数在port.c中定义，具体见代码清单7‑10。

代码清单7‑10vPortSetupTimerInterrupt()函数

1 /\* SysTick 控制寄存器 \*/**(1)**

2 #define portNVIC_SYSTICK_CTRL_REG (*((volatile uint32_t \*) 0xe000e010
))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))

3 /\* SysTick 重装载寄存器寄存器 \*/

4 #define portNVIC_SYSTICK_LOAD_REG (*((volatile uint32_t \*) 0xe000e014
))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))

5

6 /\* SysTick 时钟源选择 \*/

**7 #ifndef configSYSTICK_CLOCK_HZ**

**8 #define configSYSTICK_CLOCK_HZ configCPU_CLOCK_HZ**

**9 /\* 确保SysTick的时钟与内核时钟一致 \*/**

**10 #define portNVIC_SYSTICK_CLK_BIT ( 1UL << 2UL )**

**11 #else**

**12 #define portNVIC_SYSTICK_CLK_BIT ( 0 )**

**13 #endif**

14

15 #define portNVIC_SYSTICK_INT_BIT ( 1UL << 1UL )

16 #define portNVIC_SYSTICK_ENABLE_BIT ( 1UL << 0UL )

17

18

19 void vPortSetupTimerInterrupt( void )\ **(2)**

20 {

21 /\* 设置重装载寄存器的值 \*/**(2)-①**

22 portNVIC_SYSTICK_LOAD_REG = ( configSYSTICK_CLOCK_HZ / configTICK_RATE_HZ ) - 1UL;

23

24 /\* 设置系统定时器的时钟等于内核时钟\ **(2)-②**

25 使能SysTick 定时器中断

26 使能SysTick 定时器 \*/

27 portNVIC_SYSTICK_CTRL_REG = ( portNVIC_SYSTICK_CLK_BIT \|

28 portNVIC_SYSTICK_INT_BIT \|

29 portNVIC_SYSTICK_ENABLE_BIT );

30 }

代码清单7‑10\ **(1)**\ ：配置SysTick需要用到的寄存器和宏定义，在port.c中实现。

代码清单7‑10\ **(2)**\ ：SysTick初始化函数vPortSetupTimerInterrupt()，在xPortStartScheduler()中被调用，具体见代码清单7‑11的加粗部分。

代码清单7‑11xPortStartScheduler()函数中调用vPortSetupTimerInterrupt()

1 BaseType_t xPortStartScheduler( void )

2 {

3 /\* 配置PendSV 和 SysTick 的中断优先级为最低 \*/

4 portNVIC_SYSPRI2_REG \|= portNVIC_PENDSV_PRI;

**5 portNVIC_SYSPRI2_REG \|= portNVIC_SYSTICK_PRI;**

**6**

**7 /\* 初始化SysTick \*/**

**8 vPortSetupTimerInterrupt();**

9

10 /\* 启动第一个任务，不再返回 \*/

11 prvStartFirstTask();

12

13 /\* 不应该运行到这里 \*/

14 return 0;

15 }

代码清单7‑10\ **(2)-①**\ ：设置重装载寄存器的值，决定SysTick的中断周期。从代码清单7‑10\ **(1)**\ 可以知道：如果没有定义configSYSTICK_CLOCK_HZ那么configSYSTICK_CLOCK_HZ就等于configCPU_CLOCK_HZ，con
figSYSTICK_CLOCK_HZ确实没有定义，则configSYSTICK_CLOCK_HZ由在FreeRTOSConfig.h中定义的configCPU_CLOCK_HZ决定，同时configTICK_RATE_HZ也在FreeRTOSConfig.h中定义，具体见代码清单7‑12。

代码清单7‑12configCPU_CLOCK_HZ与configTICK_RATE_HZ宏定义

1 #define configCPU_CLOCK_HZ (( unsigned long ) 25000000)\ **(1)**

2 #define configTICK_RATE_HZ (( TickType_t ) 100)\ **(2)**

代码清单7‑12\ **(1)**\ ：系统时钟的大小，因为目前是软件仿真，需要配置成与system_ARMCM3.c(system_ARMCM4.c或system_ARMCM7.c)文件中的SYSTEM_CLOCK的一样，即等于25M。如果有具体的硬件，则配置成与硬件的系统时钟一样。

代码清单7‑12\ **(2)**\ ：SysTick每秒中断多少次，目前配置为100，即每10ms中断一次。

代码清单7‑10\ **(2)-②**\ ：设置系统定时器的时钟等于内核时钟，使能SysTick 定时器中断，使能SysTick 定时器。

main函数
~~~~~~

main函数和任务代码变动不大，具体见代码清单7‑13，有变动部分代码已加粗。

代码清单7‑13 main函数

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

17 extern List_t pxReadyTasksLists[ configMAX_PRIORITIES ];

18

19

20 /\*

21 \\*

22 \* 任务控制块& STACK

23 \\*

24 \*/

25 TaskHandle_t Task1_Handle;

26 #define TASK1_STACK_SIZE 128

27 StackType_t Task1Stack[TASK1_STACK_SIZE];

28 TCB_t Task1TCB;

29

30 TaskHandle_t Task2_Handle;

31 #define TASK2_STACK_SIZE 128

32 StackType_t Task2Stack[TASK2_STACK_SIZE];

33 TCB_t Task2TCB;

34

35

36 /\*

37 \\*

38 \* 函数声明

39 \\*

40 \*/

41 void delay (uint32_t count);

42 void Task1_Entry( void \*p_arg );

43 void Task2_Entry( void \*p_arg );

44

45 /\*

46 \\*

47 \* main函数

48 \\*

49 \*/

50

51 int main(void)

52 {

53 /\* 硬件初始化 \*/

54 /\* 将硬件相关的初始化放在这里，如果是软件仿真则没有相关初始化代码 \*/

55

56 /\* 初始化与任务相关的列表，如就绪列表 \*/

57 prvInitialiseTaskLists();

58

59 /\* 创建任务 \*/

60 Task1_Handle =

61 xTaskCreateStatic( (TaskFunction_t)Task1_Entry, /\* 任务入口 \*/

62 (char \*)"Task1", /\* 任务名称，字符串形式 \*/

63 (uint32_t)TASK1_STACK_SIZE , /\* 任务栈大小，单位为字 \*/

64 (void \*) NULL, /\* 任务形参 \*/

65 (StackType_t \*)Task1Stack, /\* 任务栈起始地址 \*/

66 (TCB_t \*)&Task1TCB ); /\* 任务控制块 \*/

67 /\* 将任务添加到就绪列表 \*/

68 vListInsertEnd( &( pxReadyTasksLists[1] ),

69 &( ((TCB_t \*)(&Task1TCB))->xStateListItem ) );

70

71 Task2_Handle =

72 xTaskCreateStatic( (TaskFunction_t)Task2_Entry, /\* 任务入口 \*/

73 (char \*)"Task2", /\* 任务名称，字符串形式 \*/

74 (uint32_t)TASK2_STACK_SIZE , /\* 任务栈大小，单位为字 \*/

75 (void \*) NULL, /\* 任务形参 \*/

76 (StackType_t \*)Task2Stack, /\* 任务栈起始地址 \*/

77 (TCB_t \*)&Task2TCB ); /\* 任务控制块 \*/

78 /\* 将任务添加到就绪列表 \*/

79 vListInsertEnd( &( pxReadyTasksLists[2] ),

80 &( ((TCB_t \*)(&Task2TCB))->xStateListItem ) );

81

82 /\* 启动调度器，开始多任务调度，启动成功则不返回 \*/

83 vTaskStartScheduler();

84

85 for (;;)

86 {

87 /\* 系统启动成功不会到达这里 \*/

88 }

89 }

90

91 /\*

92 \\*

93 \* 函数实现

94 \\*

95 \*/

96 /\* 软件延时 \*/

97 void delay (uint32_t count)

98 {

99 for (; count!=0; count--);

100 }

101 /\* 任务1 \*/

102 void Task1_Entry( void \*p_arg )

103 {

104 for ( ;; )

105 {

106 #if 0

107 flag1 = 1;

108 delay( 100 );

109 flag1 = 0;

110 delay( 100 );

111

112 /\* 任务切换，这里是手动切换 \*/

113 portYIELD();

114 #else

**115 flag1 = 1;**

**116 vTaskDelay( 2 );(1)**

**117 flag1 = 0;**

**118 vTaskDelay( 2 );**

119 #endif

120 }

121 }

122

123 /\* 任务2 \*/

124 void Task2_Entry( void \*p_arg )

125 {

126 for ( ;; )

127 {

128 #if 0

129 flag2 = 1;

130 delay( 100 );

131 flag2 = 0;

132 delay( 100 );

133

134 /\* 任务切换，这里是手动切换 \*/

135 portYIELD();

136 #else

**137 flag2 = 1;**

**138 vTaskDelay( 2 );(2)**

**139 flag2 = 0;**

**140 vTaskDelay( 2 );**

141 #endif

142 }

143 }

144

**145 /\* 获取空闲任务的内存 \*/**

**146 StackType_t IdleTaskStack[configMINIMAL_STACK_SIZE];(3)**

**147 TCB_t IdleTaskTCB;**

**148 void vApplicationGetIdleTaskMemory( TCB_t \**ppxIdleTaskTCBBuffer,**

**149 StackType_t \**ppxIdleTaskStackBuffer,**

**150 uint32_t \*pulIdleTaskStackSize )**

**151 {**

**152 \*ppxIdleTaskTCBBuffer=&IdleTaskTCB;**

**153 \*ppxIdleTaskStackBuffer=IdleTaskStack;**

**154 \*pulIdleTaskStackSize=configMINIMAL_STACK_SIZE;**

**155 }**

代码清单7‑13\ **(1)**\ 和\ **(2)**\ ：延时函数均由原来的软件延时替代为阻塞延时，延时时间均为2个SysTick中断周期，即20ms。

代码清单7‑13\ **(3)**\ ：定义空闲任务的栈和TCB。

实验现象
~~~~

进入软件调试，全速运行程序，从逻辑分析仪中可以看到两个任务的波形是完全同步，就好像CPU在同时干两件事情，具体仿真的波形图见图7‑1和图7‑2。

|idleth002|

图7‑1实验现象1

|idleth003|

图7‑2实验现象2

从图7‑1和图7‑2可以看出，flag1和flag2的高电平的时间为(0.1802-0.1602)s，刚好等于阻塞延时的20ms，所以实验现象跟代码要实现的功能是一致的。

.. |idleth002| image:: media\idleth002.png
   :width: 4.53472in
   :height: 2.02441in
.. |idleth003| image:: media\idleth003.png
   :width: 4.48611in
   :height: 2.32731in
