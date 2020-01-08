.. vim: syntax=rst

支持时间片
-----

FreeRTOS与隔壁的RT-Thread和μC/OS一样，都支持时间片的功能。所谓时间片就是同一个优先级下可以有多个任务，每个任务轮流地享有相同的CPU时间，享有CPU的时间我们叫时间片。在RTOS中，最小的时间单位为一个tick，即SysTick的中断周期，RT-
Thread和μC/OS可以指定时间片的大小为多个tick，但是FreeRTOS不一样，时间片只能是一个tick。与其说FreeRTOS支持时间片，倒不如说它的时间片就是正常的任务调度。

其实时间片的功能我们已经实现，剩下的就是通过实验来验证。那么接下来我们就先看实验现象，再分析原理，透过现象看本质。

时间片测试实验
~~~~~~~

假设目前系统中有三个任务就绪（算上空闲任务就是4个），任务1和任务2的优先级为2，任务3的优先级为3，整个就绪列表的示意图具体见图10‑1。

|slidin002|

图10‑1有三个任务就绪时就绪列表示意图（空闲任务没有画出来）

为了方便在逻辑分析仪中地分辨出任务1和任务2使用的时间片大小，任务1和任务2的主体编写成一个无限循环函数，不会阻塞，任务3的阻塞时间设置为1个tick。任务1和任务2的任务主体编写为一个无限循环，这就意味着，优先级低于2的任务就会被饿死，得不到执行，比如空闲任务。在真正的项目中，并不会这样写，这里只
是为了实验方便。整个mai.c的文件的实验代码具体见代码清单10‑1。

main函数
~~~~~~

代码清单10‑1时间片实验

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

16 portCHAR flag3;

17

18 extern List_t pxReadyTasksLists[ configMAX_PRIORITIES ];

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

**35 TaskHandle_t Task3_Handle;**

**36 #define TASK3_STACK_SIZE 128**

**37 StackType_t Task3Stack[TASK3_STACK_SIZE];**

**38 TCB_t Task3TCB;**

39

40 /\*

41 \\*

42 \* 函数声明

43 \\*

44 \*/

45 void delay (uint32_t count);

46 void Task1_Entry( void \*p_arg );

47 void Task2_Entry( void \*p_arg );

48 void Task3_Entry( void \*p_arg );

49

50

51 /\*

52 \\*

53 \* main函数

54 \\*

55 \*/

56 int main(void)

57 {

58 /\* 硬件初始化 \*/

59 /\* 将硬件相关的初始化放在这里，如果是软件仿真则没有相关初始化代码 \*/

60

61 /\* 创建任务 \*/

62 Task1_Handle =

63 xTaskCreateStatic( (TaskFunction_t)Task1_Entry,

64 (char \*)"Task1",

65 (uint32_t)TASK1_STACK_SIZE ,

66 (void \*) NULL,

**67 (UBaseType_t) 2,**

68 (StackType_t \*)Task1Stack,

69 (TCB_t \*)&Task1TCB );

70

71 Task2_Handle =

72 xTaskCreateStatic( (TaskFunction_t)Task2_Entry,

73 (char \*)"Task2",

74 (uint32_t)TASK2_STACK_SIZE ,

75 (void \*) NULL,

**76 (UBaseType_t) 2,**

77 (StackType_t \*)Task2Stack,

78 (TCB_t \*)&Task2TCB );

79

80 Task3_Handle =

81 xTaskCreateStatic( (TaskFunction_t)Task3_Entry,

82 (char \*)"Task3",

83 (uint32_t)TASK3_STACK_SIZE ,

84 (void \*) NULL,

**85 (UBaseType_t) 3,**

86 (StackType_t \*)Task3Stack,

87 (TCB_t \*)&Task3TCB );

88

89 portDISABLE_INTERRUPTS();

90

91 /\* 启动调度器，开始多任务调度，启动成功则不返回 \*/

92 vTaskStartScheduler();\ **(1)**

93

94 for (;;)

95 {

96 /\* 系统启动成功不会到达这里 \*/

97 }

98 }

99

100 /\*

101 \\*

102 \* 函数实现

103 \\*

104 \*/

105 /\* 软件延时 \*/

106 void delay (uint32_t count)

107 {

108 for (; count!=0; count--);

109 }

**110 /\* 任务1 \*/(2)**

**111 void Task1_Entry( void \*p_arg )**

**112 {**

**113 for ( ;; )**

**114 {**

**115 flag1 = 1;**

**116 //vTaskDelay( 1 );**

**117 delay (100);**

**118 flag1 = 0;**

**119 delay (100);**

**120 //vTaskDelay( 1 );**

**121 }**

**122 }**

123

**124 /\* 任务2 \*/(3)**

**125 void Task2_Entry( void \*p_arg )**

**126 {**

**127 for ( ;; )**

**128 {**

**129 flag2 = 1;**

**130 //vTaskDelay( 1 );**

**131 delay (100);**

**132 flag2 = 0;**

**133 delay (100);**

**134 //vTaskDelay( 1 );**

**135 }**

**136 }**

137

138

139 void Task3_Entry( void \*p_arg )\ **(4)**

140 {

141 for ( ;; )

142 {

143 flag3 = 1;

144 vTaskDelay( 1 );

145 //delay (100);

146 flag3 = 0;

147 vTaskDelay( 1 );

148 //delay (100);

149 }

150 }

151

152 /\* 获取空闲任务的内存 \*/

153 StackType_t IdleTaskStack[configMINIMAL_STACK_SIZE];

154 TCB_t IdleTaskTCB;

155 void vApplicationGetIdleTaskMemory( TCB_t \**ppxIdleTaskTCBBuffer,

156 StackType_t \**ppxIdleTaskStackBuffer,

157 uint32_t \*pulIdleTaskStackSize )

158 {

159 \*ppxIdleTaskTCBBuffer=&IdleTaskTCB;

160 \*ppxIdleTaskStackBuffer=IdleTaskStack;

161 \*pulIdleTaskStackSize=configMINIMAL_STACK_SIZE;

162 }

代码清单10‑1\ **(2)和(3)**\ ：为了方便观察任务1和任务2使用的时间片大小，特意将任务的主体编写成一个无限循环。实际项目中不会这样使用，否则低于任务1和任务2优先级的任务就会被饿死，一直没有机会被执行。

代码清单10‑1\ **(4)**\ ：因为任务1和任务2的主体是无限循环的，要想任务3有机会执行，其优先级就必须高于任务1和任务2的优先级。为了方便观察任务1和任务2使用的时间片大小，任务3的阻塞延时我们设置为1个tick。

实验现象
~~~~

进入软件调试，全速运行程序，从逻辑分析仪中可以看到任务1和任务2轮流执行，每一次运行的时间等于任务3中flag3输出高电平或者低电平的时间，即一个tick，具体仿真的波形图见图10‑2。

|slidin003|

图10‑2时间片实验实验现象

在这一个tick（时间片）里面，任务1和任务2的flag标志位做了很多次的翻转，点击逻辑分析仪中Zoom In 按钮将波形放大后就可以看到flag翻转的细节，具体见图10‑3。

|slidin004|

图10‑3任务中flag翻转的细节图

原理分析
~~~~

之所以在同一个优先级下可以有多个任务，最终还是得益于taskRESET_READY_PRIORITY()和taskSELECT_HIGHEST_PRIORITY_TASK()这两个函函数的实现方法。接下来我们分析下这两个函数是如何在同一个优先级下有多个任务的时候起作用的。

系统在任务切换的时候总会从就绪列表中寻找优先级最高的任务来执行，寻找优先级最高的任务这个功能由taskSELECT_HIGHEST_PRIORITY_TASK()函数来实现，该函数在task.c中定义，具体实现见代码清单10‑2。

taskSELECT_HIGHEST_PRIORITY_TASK()函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

代码清单10‑2taskSELECT_HIGHEST_PRIORITY_TASK()函数

1 #define taskSELECT_HIGHEST_PRIORITY_TASK()\\

2 {\\

3 UBaseType_t uxTopPriority;\\

4 /\* 寻找就绪任务的最高优先级 \*/\\\ **(1)**

5 portGET_HIGHEST_PRIORITY( uxTopPriority, uxTopReadyPriority );\\

6 /\* 获取优先级最高的就绪任务的TCB，然后更新到pxCurrentTCB \*/\\\ **(2)**

7 listGET_OWNER_OF_NEXT_ENTRY( pxCurrentTCB,\\

8 &( pxReadyTasksLists[ uxTopPriority ] ) );\\

9 }

代码清单10‑2\ **(1)**\ ：寻找就绪任务的最高优先级。即根据优先级位图表uxTopReadyPriority找到就绪任务的最高优先级，然后将优先级暂存在uxTopPriority。

代码清单10‑2\ **(2)**\ ：获取优先级最高的就绪任务的TCB，然后更新到pxCurrentTCB。目前我们的实验是在优先级2上有任务1和任务2，假设任务1运行了一个tick，那接下来再从对应优先级2的就绪列表上选择任务来运行就应该是选择任务2？怎么选择，代码上怎么实现？奥妙就在listG
ET_OWNER_OF_NEXT_ENTRY()函数中，该函数在list.h中定义，具体实现见代码清单10‑3。

代码清单10‑3listGET_OWNER_OF_NEXT_ENTRY()函数

1 #define listGET_OWNER_OF_NEXT_ENTRY( pxTCB, pxList )\\

2 {\\

3 List_t \* const pxConstList = ( pxList );\\

4 /\* 节点索引指向链表第一个节点调整节点索引指针，指向下一个节点，

5 如果当前链表有N个节点，当第N次调用该函数时，pxIndex则指向第N个节点 \*/\\

6 ( pxConstList )->pxIndex = ( pxConstList )->pxIndex->pxNext;\\

7 /\* 当遍历完链表后，pxIndex回指到根节点 \*/\\

8 if( ( void \* ) ( pxConstList )->pxIndex == ( void \* ) &( ( pxConstList )->xListEnd ) )\\

9 {\\

10 ( pxConstList )->pxIndex = ( pxConstList )->pxIndex->pxNext;\\

11 }\\

12 /\* 获取节点的OWNER，即TCB \*/\\

13 ( pxTCB ) = ( pxConstList )->pxIndex->pvOwner;\\

14 }

listGET_OWNER_OF_NEXT_ENTRY()函数的妙处在于它并不是获取链表下的第一个节点的OWNER，而且用于获取下一个节点的OWNER。有下一个那么就会有上一个的说法，怎么理解？假设当前链表有N个节点，当第N次调用该函数时，pxIndex则指向第N个节点，即每调用一次，节点遍历指针p
xIndex则会向后移动一次，用于指向下一个节点。

本实验中，优先级2下有两个任务，当系统第一次切换到优先级为2的任务（包含了任务1和任务2，因为它们的优先级相同）时，pxIndex指向任务1，任务1得到执行。当任务1执行完毕，系统重新切换到优先级为2的任务时，这个时候pxIndex指向任务2，任务2得到执行，任务1和任务2轮流执行，享有相同的CPU
时间，即所谓的时间片。

本实验中，任务1和任务2的主体都是无限循环，那如果任务1和任务2都会调用将自己挂起的函数（实际运用中，任务体都不能是无限循环的，必须调用能将自己挂起的函数），比如vTaskDelay()。调用能将任务挂起的函数中，都会先将任务从就绪列表删除，然后将任务在优先级位图表uxTopReadyPriorit
y中对应的位清零，这一功能由taskRESET_READY_PRIORITY()函数来实现，该函数在task.c中定义，具体实现见代码清单10‑4。

taskRESET_READY_PRIORITY()函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

代码清单10‑4taskRESET_READY_PRIORITY()函数

1 #define taskRESET_READY_PRIORITY( uxPriority )\\

2 {\\

**3 if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ ( uxPriority ) ] ) )\\**

**4 == ( UBaseType_t ) 0 )\\**

5 {\\

6 portRESET_READY_PRIORITY( ( uxPriority ),\\

7 ( uxTopReadyPriority ) );\\

8 }\\

9 }

taskRESET_READY_PRIORITY()函数的妙处在于清除优先级位图表uxTopReadyPriority中相应的位时候，会先判断当前优先级链表下是否还有其他任务，如果有则不清零。假设当前实验中，任务1会调用vTaskDelay()，会将自己挂起，只能是将任务1从就绪列表删除，不能将任务
1在优先级位图表uxTopReadyPriority中对应的位清0，因为该优先级下还有任务2，否则任务2将得不到执行。

修改代码，支持优先级
~~~~~~~~~~

其实，我们的代码已经支持了时间片，实现的算法与FreeRTOS官方是一样的，即taskSELECT_HIGHEST_PRIORITY_TASK()和taskRESET_READY_PRIORITY()这两个函数的实现。但是在代码的编排组织上与FreeRTOS官方的还是有点不一样，为了与FreeRTO
S官方代码统一起来，我们还是稍作修改。

xPortSysTickHandler()函数
^^^^^^^^^^^^^^^^^^^^^^^

xPortSysTickHandler()函数具体修改见代码清单10‑5的加粗部分，即当xTaskIncrementTick()函数返回为真时才进行任务切换，原来的xTaskIncrementTick()是不带返回值的，执行到最后会调用taskYIELD()执行任务切换。

代码清单10‑5xPortSysTickHandler()函数

1 void xPortSysTickHandler( void )

2 {

3 /\* 关中断 \*/

4 vPortRaiseBASEPRI();

5

6 {

**7 //xTaskIncrementTick();**

**8**

**9 /\* 更新系统时基 \*/**

**10 if ( xTaskIncrementTick() != pdFALSE )**

**11 {**

**12 /\* 任务切换，即触发PendSV \*/**

**13 //portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;**

**14 taskYIELD();**

**15 }**

16 }

17

18 /\* 开中断 \*/

19 vPortClearBASEPRIFromISR();

20 }

修改xTaskIncrementTick()函数
''''''''''''''''''''''''

xTaskIncrementTick()函数具体修改见代码清单10‑6的加粗部分。

代码清单10‑6xTaskIncrementTick()函数

**1 //void xTaskIncrementTick( void )**

**2 BaseType_t xTaskIncrementTick( void )(1)**

3 {

4 TCB_t \* pxTCB;

5 TickType_t xItemValue;

**6 BaseType_t xSwitchRequired = pdFALSE;(2)**

7

8 const TickType_t xConstTickCount = xTickCount + 1;

9 xTickCount = xConstTickCount;

10

11 /\* 如果xConstTickCount溢出，则切换延时列表 \*/

12 if ( xConstTickCount == ( TickType_t ) 0U )

13 {

14 taskSWITCH_DELAYED_LISTS();

15 }

16

17 /\* 最近的延时任务延时到期 \*/

18 if ( xConstTickCount >= xNextTaskUnblockTime )

19 {

20 for ( ;; )

21 {

22 if ( listLIST_IS_EMPTY( pxDelayedTaskList ) != pdFALSE )

23 {

24 /\* 延时列表为空，设置xNextTaskUnblockTime为可能的最大值 \*/

25 xNextTaskUnblockTime = portMAX_DELAY;

26 break;

27 }

28 else/\* 延时列表不为空 \*/

29 {

30 pxTCB = ( TCB_t \* ) listGET_OWNER_OF_HEAD_ENTRY( pxDelayedTaskList );

31 xItemValue = listGET_LIST_ITEM_VALUE( &( pxTCB->xStateListItem ) );

32

33 /\* 直到将延时列表中所有延时到期的任务移除才跳出for循环 \*/

34 if ( xConstTickCount < xItemValue )

35 {

36 xNextTaskUnblockTime = xItemValue;

37 break;

38 }

39

40 /\* 将任务从延时列表移除，消除等待状态 \*/

41 ( void ) uxListRemove( &( pxTCB->xStateListItem ) );

42

43 /\* 将解除等待的任务添加到就绪列表 \*/

44 prvAddTaskToReadyList( pxTCB );

45

46

**47 #if ( configUSE_PREEMPTION == 1 )(3)**

**48 {**

**49 if ( pxTCB->uxPriority >= pxCurrentTCB->uxPriority )**

**50 {**

**51 xSwitchRequired = pdTRUE;**

**52 }**

**53 }**

**54 #endif/\* configUSE_PREEMPTION \*/**

55 }

56 }

57 }/\* xConstTickCount >= xNextTaskUnblockTime \*/

58

**59 #if ( ( configUSE_PREEMPTION == 1 ) && ( configUSE_TIME_SLICING == 1 ) )(4)**

**60 {**

**61 if ( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ pxCurrentTCB->uxPriority ] ) )**

**62 > ( UBaseType_t ) 1 )**

**63 {**

**64 xSwitchRequired = pdTRUE;**

**65 }**

**66 }**

**67 #endif/\* ( ( configUSE_PREEMPTION == 1 ) && ( configUSE_TIME_SLICING == 1 ) ) \*/**

68

69

**70 /\* 任务切换 \*/**

**71 //portYIELD();(5)**

72 }

代码清单10‑6\ **(1)**\ ：将xTaskIncrementTick()函数修改成带返回值的函数。

代码清单10‑6\ **(2)**\ ：定义一个局部变量xSwitchRequired，用于存储xTaskIncrementTick()函数的返回值，当返回值是pdTRUE时，需要执行一次任务切换，默认初始化为pdFALSE。

代码清单10‑6\ **(3)**\ ：configUSE_PREEMPTION是在FreeRTOSConfig.h的一个宏，默认为1，表示有任务就绪且就绪任务的优先级比当前优先级高时，需要执行一次任务切换，即将xSwitchRequired的值置为pdTRUE。在xTaskIncrementTic
k()函数还没有修改成带返回值的时候，我们是在执行完xTaskIncrementTick()函数的时候，不管是否有任务就绪，不管就绪的任务的优先级是否比当前任务优先级高都执行一次任务切换。如果就绪任务的优先级比当前优先级高，那么执行一次任务切换与加了代码清单10‑6\ **(3)**\
这段代码实现的功能是一样的。如果没有任务就绪呢？就不需要执行任务切换，这样与之前的实现方法相比就省了一次任务切换的时间。虽然说没有更高优先级的任务就绪，执行任务切换的时候还是会运行原来的任务，但这是以多花一次任务切换的时间为代价的。

代码清单10‑6\ **(4)**\ ：这部分与时间片功能相关。当configUSE_PREEMPTION与configUSE_TIME_SLICING都为真，且当前优先级下不止一个任务时就执行一次任务切换，即将xSwitchRequired置为pdTRUE即可。在xTaskIncrementTic
k()函数还没有修改成带返回值之前，这部分代码不需要也是可以实现时间片功能的，即只要在执行完xTaskIncrementTick()函数后执行一次任务切换即可。configUSE_PREEMPTION在FreeRTOSConfig.h中默认定义为1，configUSE_TIME_SLICING如果没
有定义，则会默认在FreeRTOS.h中定义为1。

其实FreeRTOS的这种时间片功能不能说是真正意义的时间片，因为它不能随意的设置时间为多少个tick，而是默认一个tick，然后默认在每个tick中断周期中进行任务切换而已。

代码清单10‑6\ **(5)**\ ：不在这里进行任务切换，而是放到了xPortSysTickHandler()函数中。当xTaskIncrementTick()函数的返回值为真时才进行任务切换。

至此，FreeRTOS时间片功能就讲完。本书第一部分的知识点“从0到1教你写FreeRTOS内核”也就到这里完结。

.. |slidin002| image:: media\slidin002.png
   :width: 5.76806in
   :height: 3.37361in
.. |slidin003| image:: media\slidin003.png
   :width: 5.76806in
   :height: 2.65771in
.. |slidin004| image:: media\slidin004.png
   :width: 5.76806in
   :height: 1.92803in
