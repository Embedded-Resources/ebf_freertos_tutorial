.. vim: syntax=rst

创建任务
=========

在上一章，我们已经基于野火STM32开发板创建好了FreeRTOS的工程模板，这章开始我们将真正进入如何使用FreeRTOS的征程，先从最简单的创建任务开始，点亮一个LED，以慰藉下尔等初学者弱小的心灵。

硬件初始化
~~~~~

本章创建的任务需要用到开发板上的LED，所以先要将LED相关的函数初始化好，为了方便以后统一管理板级外设的初始化，我们在main.c文件中创建一个BSP_Init()函数，专门用于存放板级外设初始化函数，具体见代码清单12‑1的加粗部分。

代码清单12‑1 BSP_Init()中添加硬件初始化函数

1 /\*

2 \* @ 函数名： BSP_Init

3 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

4 \* @ 参数：NULL

5 \* @ 返回值：无

6 \/

**7 static void BSP_Init(void)**

**8 {**

**9 /\**

**10 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15**

**11 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，**

**12 \* 都统一用这个优先级分组，千万不要再分组，切忌。**

**13 \*/**

**14 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );**

**15**

**16 /\* LED 初始化 \*/**

**17 LED_GPIO_Config();**

**18**

**19 /\* 串口初始化 \*/**

**20 USART_Config();**

**21**

**22 }**

执行到BSP_Init()函数的时候，操作系统完全都还没有涉及，即BSP_Init()函数所做的工作跟我们以前编写的裸机工程里面的硬件初始化工作是一模一样的。运行完BSP_Init ()函数，接下来才慢慢启动操作系统，最后运行创建好的任务。有时候任务创建好，整个系统跑起来了，可想要的实验现象就是出不
来，比如LED不会亮，串口没有输出，LCD没有显示等等。如果是初学者，这个时候就会心急如焚，四处求救，那怎么办？这个时候如何排除是硬件的问题还是系统的问题，这里面有个小小的技巧，即在硬件初始化好之后，顺便测试下硬件，测试方法跟裸机编程一样，具体实现见代码清单12‑2的加粗部分。

代码清单12‑2BSP_Init()中添加硬件测试函数

1 /\* 开发板硬件bsp头文件 \*/

2 #include"bsp_led.h"

3 #include"bsp_usart.h"

4

5 /\*

6 \* @ 函数名： BSP_Init

7 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

8 \* @ 参数：

9 \* @ 返回值：无

10 \/

11 static void BSP_Init(void)

12 {

13 /\*

14 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

15 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

16 \* 都统一用这个优先级分组，千万不要再分组，切忌。

17 \*/

18 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

19

20 /\* LED 初始化 \*/

21 LED_GPIO_Config(); **(1)**

22

23 /\* 测试硬件是否正常工作 \*/ **(2)**

24 LED1_ON;

25

26 /\* 其他硬件初始化和测试 \*/

27

28 /\* 让程序停在这里，不再继续往下执行 \*/

29 while (1); **(3)**

30

31 /\* 串口初始化 \*/

32 USART_Config();

33

34 }

代码清单12‑2\ **(1)**\ ：初始化硬件后，顺便测试硬件，看下硬件是否正常工作。

代码清单12‑2\ **(2)**\ ：可以继续添加其他的硬件初始化和测试。硬件确认没有问题之后，硬件测试代码可删可不删，因为BSP_Init()函数只执行一遍。

代码清单12‑2\ **(3)**\ ：方便测试硬件好坏，让程序停在这里，不再继续往下执行，当测试完毕后，这个while(1);必须删除。

创建单任务—SRAM静态内存
~~~~~~~~~~~~~~

这里，我们创建一个单任务，任务使用的栈和任务控制块都使用静态内存，即预先定义好的全局变量，这些预先定义好的全局变量都存在内部的SRAM中。

定义任务函数
^^^^^^

任务实际上就是一个无限循环且不带返回值的C函数。目前，我们创建一个这样的任务，让开发板上面的LED灯以500ms的频率闪烁，具体实现见代码清单12‑3。

代码清单12‑320.6.1 定义任务函数

1 static voidLED_Task (void\* parameter)

2 {

3 while (1) **(1)**

4 {

5 LED1_ON;

6 vTaskDelay(500); /\* 延时500个tick \*/**(2)**

7

8 LED1_OFF;

9 vTaskDelay(500); /\* 延时500个tick \*/

10

11 }

12 }

代码清单12‑3\ **(1)**\ ：任务必须是一个死循环，否则任务将通过LR返回，如果LR指向了非法的内存就会产生HardFault_Handler，而FreeRTOS指向一个死循环，那么任务返回之后就在死循环中执行，这样子的任务是不安全的，所以避免这种情况，任务一般都是死循环并且无返回值的。我
们的AppTaskCreate任务，执行一次之后就进行删除，则不影响系统运行，所以，只执行一次的任务在执行完毕要记得及时删除。

代码清单12‑3\ **(2)**\ ：任务里面的延时函数必须使用FreeRTOS里面提供的延时函数，并不能使用我们裸机编程中的那种延时。这两种的延时的区别是FreeRTOS里面的延时是阻塞延时，即调用vTaskDelay()函数的时候，当前任务会被挂起，调度器会切换到其他就绪的任务，从而实现多任务
。如果还是使用裸机编程中的那种延时，那么整个任务就成为了一个死循环，如果恰好该任务的优先级是最高的，那么系统永远都是在这个任务中运行，比它优先级更低的任务无法运行，根本无法实现多任务。

空闲任务与定时器任务栈函数实现
^^^^^^^^^^^^^^^

当我们使用了静态创建任务的时候，configSUPPORT_STATIC_ALLOCATION这个宏定义必须为1（在FreeRTOSConfig.h文件中），并且我们需要实现两个函数：vApplicationGetIdleTaskMemory()与vApplicationGetTimerTaskMe
mory()，这两个函数是用户设定的空闲（Idle）任务与定时器（Timer）任务的栈大小，必须由用户自己分配，而不能是动态分配，具体见代码清单12‑4加粗部分。

代码清单12‑4空闲任务与定时器任务栈函数实现

**1 /\* 空闲任务任务栈 \*/**

**2 static StackType_t Idle_Task_Stack[configMINIMAL_STACK_SIZE];**

**3 /\* 定时器任务栈 \*/**

**4 static StackType_t Timer_Task_Stack[configTIMER_TASK_STACK_DEPTH];**

**5**

**6 /\* 空闲任务控制块 \*/**

**7 static StaticTask_t Idle_Task_TCB;**

**8 /\* 定时器任务控制块 \*/**

**9 static StaticTask_t Timer_Task_TCB;**

10

11 /*\*

12 \\*

13 \* @brief 获取空闲任务的任务栈和任务控制块内存

14 \*ppxTimerTaskTCBBuffer : 任务控制块内存

15 \*ppxTimerTaskStackBuffer : 任务栈内存

16 \*pulTimerTaskStackSize : 任务栈大小

17 \* @author fire

18 \* @version V1.0

19 \* @date 2018-xx-xx

20 \\*

21 \*/

**22 void vApplicationGetIdleTaskMemory(StaticTask_t \**ppxIdleTaskTCBBuffer,**

**23 StackType_t \**ppxIdleTaskStackBuffer,**

**24 uint32_t \*pulIdleTaskStackSize)**

**25 {**

**26 \*ppxIdleTaskTCBBuffer=&Idle_Task_TCB;/\* 任务控制块内存 \*/**

**27 \*ppxIdleTaskStackBuffer=Idle_Task_Stack;/\* 任务栈内存 \*/**

**28 \*pulIdleTaskStackSize=configMINIMAL_STACK_SIZE;/\* 任务栈大小 \*/**

**29 }**

30

31 /*\*

32 \\*

33 \* @brief 获取定时器任务的任务栈和任务控制块内存

34 \*ppxTimerTaskTCBBuffer : 任务控制块内存

35 \*ppxTimerTaskStackBuffer : 任务栈内存

36 \*pulTimerTaskStackSize : 任务栈大小

37 \* @author fire

38 \* @version V1.0

39 \* @date 2018-xx-xx

40 \\*

41 \*/

**42 void vApplicationGetTimerTaskMemory(StaticTask_t \**ppxTimerTaskTCBBuffer,**

**43 StackType_t \**ppxTimerTaskStackBuffer,**

**44 uint32_t \*pulTimerTaskStackSize)**

**45 {**

**46 \*ppxTimerTaskTCBBuffer=&Timer_Task_TCB;/\* 任务控制块内存 \*/**

**47 \*ppxTimerTaskStackBuffer=Timer_Task_Stack;/\* 任务栈内存 \*/**

**48 \*pulTimerTaskStackSize=configTIMER_TASK_STACK_DEPTH;/\* 任务栈大小 \*/**

**49 }**

定义任务栈
^^^^^

目前我们只创建了一个任务，当任务进入延时的时候，因为没有另外就绪的用户任务，那么系统就会进入空闲任务，空闲任务是FreeRTOS系统自己启动的一个任务，优先级最低。当整个系统都没有就绪任务的时候，系统必须保证有一个任务在运行，空闲任务就是为这个设计的。当用户任务延时到期，又会从空闲任务切换回用户任务
。

在FreeRTOS系统中，每一个任务都是独立的，他们的运行环境都单独的保存在他们的栈空间当中。那么在定义好任务函数之后，我们还要为任务定义一个栈，目前我们使用的是静态内存，所以任务栈是一个独立的全局变量，具体见代码清单12‑5。任务的栈占用的是MCU内部的RAM，当任务越多的时候，需要使用的栈空间就
越大，即需要使用的RAM空间就越多。一个MCU能够支持多少任务，就得看你的RAM空间有多少。

代码清单12‑5定义任务栈

1 /\* AppTaskCreate任务任务栈 \*/

2 static StackType_t AppTaskCreate_Stack[128];

3

4 /\* LED任务栈 \*/

5 static StackType_t LED_Task_Stack[128];

在大多数系统中需要做栈空间地址对齐，在FreeRTOS中是以8字节大小对齐，并且会检查栈是否已经对齐，其中portBYTE_ALIGNMENT是在portmacro.h里面定义的一个宏，其值为8，就是配置为按8字节对齐，当然用户可以选择按1、2、4、8、16、32等字节对齐，目前默认为8，具体见代码
清单12‑6。

代码清单12‑6栈空间地址对齐实现

1 #define portBYTE_ALIGNMENT 8

2

3 #if portBYTE_ALIGNMENT == 8

4 #define portBYTE_ALIGNMENT_MASK ( 0x0007 )

5 #endif

6

7 pxTopOfStack = pxNewTCB->pxStack + ( ulStackDepth - ( uint32_t ) 1 );

8 pxTopOfStack = ( StackType_t \* ) ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack ) &

9 ( ~( ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) ) );

10

11 /\* 检查计算出的栈顶部的对齐方式是否正确。 \*/

12 configASSERT( ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack &

13 ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) == 0UL ) );

定义任务控制块
^^^^^^^

定义好任务函数和任务栈之后，我们还需要为任务定义一个任务控制块，通常我们称这个任务控制块为任务的身份证。在C代码上，任务控制块就是一个结构体，里面有非常多的成员，这些成员共同描述了任务的全部信息，具体见代码清单12‑7。

代码清单12‑7定义任务控制块

1 /\* AppTaskCreate 任务控制块 \*/

2 static StaticTask_t AppTaskCreate_TCB;

3 /\* AppTaskCreate 任务控制块 \*/

4 static StaticTask_t LED_Task_TCB;

静态创建任务
^^^^^^

一个任务的三要素是任务主体函数，任务栈，任务控制块，那么怎么样把这三个要素联合在一起？FreeRTOS里面有一个叫静态任务创建函数xTaskCreateStatic()，它就是干这个活的。它将任务主体函数，任务栈（静态的）和任务控制块（静态的）这三者联系在一起，让任务可以随时被系统启动，具体见代码清
单12‑8。

代码清单12‑8静态创建任务

1 /\* 创建 AppTaskCreate 任务 \*/

2 AppTaskCreate_Handle = xTaskCreateStatic((TaskFunction_t)AppTaskCreate, //任务函数\ **(1)**

3 (const char\* )"AppTaskCreate",//任务名称\ **(2)**

4 (uint32_t )128, //任务栈大小 **(3)**

5 (void\* )NULL, //传递给任务函数的参数\ **(4)**

6 (UBaseType_t )3, //任务优先级 **(5)**

7 (StackType_t\* )AppTaskCreate_Stack, //任务栈\ **(6)**

8 (StaticTask_t\* )&AppTaskCreate_TCB); //任务控制块\ **(7)**

9

10 if (NULL != AppTaskCreate_Handle) /\* 创建成功 \*/

11 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

代码清单12‑8\ **(1)**\ ：任务入口函数，即任务函数的名称，需要我们自己定义并且实现。

代码清单12‑8\ **(2)**\ ：任务名字，字符串形式，最大长度由FreeRTOSConfig.h中定义的configMAX_TASK_NAME_LEN宏指定，多余部分会被自动截掉，这里任务名字最好要与任务函数入口名字一致，方便进行调试。

代码清单12‑8\ **(3)**\ ：任务栈大小，单位为字，在32位的处理器下（STM32），一个字等于4个字节，那么任务大小就为128 \* 4字节。

代码清单12‑8\ **(4)**\ ：任务入口函数形参，不用的时候配置为0或者NULL即可。

代码清单12‑8\ **(5)**\ ：任务的优先级。优先级范围根据FreeRTOSConfig.h中的宏configMAX_PRIORITIES决定，如果使能configUSE_PORT_OPTIMISED_TASK_SELECTION，这个宏定义，则最多支持32个优先级；如果不用特殊方法查找下一
个运行的任务，那么则不强制要求限制最大可用优先级数目。在FreeRTOS中，数值越大优先级越高，0代表最低优先级。

代码清单12‑8\ **(6)**\ ：任务栈起始地址，只有在使用静态内存的时候才需要提供，在使用动态内存的时候会根据提供的任务栈大小自动创建。

代码清单12‑8\ **(7)**\
：任务控制块指针，在使用静态内存的时候，需要给任务初始化函数xTaskCreateStatic()传递预先定义好的任务控制块的指针。在使用动态内存的时候，任务创建函数xTaskCreate()会返回一个指针指向任务控制块，该任务控制块是xTaskCreate()函数里面动态分配的一块内存。

启动任务
^^^^

当任务创建好后，是处于任务就绪（Ready），在就绪态的任务可以参与操作系统的调度。但是此时任务仅仅是创建了，还未开启任务调度器，也没创建空闲任务与定时器任务（如果使能了configUSE_TIMERS这个宏定义），那这两个任务就是在启动任务调度器中实现，每个操作系统，任务调度器只启动一次，之后就不
会再次执行了，FreeRTOS中启动任务调度器的函数是vTaskStartScheduler()，并且启动任务调度器的时候就不会返回，从此任务管理都由FreeRTOS管理，此时才是真正进入实时操作系统中的第一步，具体见代码清单12‑9。

代码清单12‑9启动任务

/\* 启动任务，开启调度 \*/

1 vTaskStartScheduler();

main.c文件内容全貌
^^^^^^^^^^^^

现在我们把任务主体，任务栈，任务控制块这三部分代码统一放到main.c中，我们在main.c文件中创建一个AppTaskCreate任务，这个任务是用于创建用户任务，为了方便管理，我们的所有的任务创建都统一放在这个函数中，在这个函数中创建成功的任务就可以直接参与任务调度了，具体内容见代码清单12‑1
0。

代码清单12‑10 main.c文件内容全貌

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS v9.0.0 + STM32 工程模版

8 \\*

9 \* @attention

10 \*

11 \* 实验平台:野火 STM32开发板

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

29

30 /\* 任务句柄 \/

31 /\*

32 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

33 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

34 \* 这个句柄可以为NULL。

35 \*/

36 /\* 创建任务句柄 \*/

37 static TaskHandle_t AppTaskCreate_Handle;

38 /\* LED任务句柄 \*/

39 static TaskHandle_t LED_Task_Handle;

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

59 /\* AppTaskCreate任务任务栈 \*/

60 static StackType_t AppTaskCreate_Stack[128];

61 /\* LED任务栈 \*/

62 static StackType_t LED_Task_Stack[128];

63

64 /\* AppTaskCreate 任务控制块 \*/

65 static StaticTask_t AppTaskCreate_TCB;

66 /\* AppTaskCreate 任务控制块 \*/

67 static StaticTask_t LED_Task_TCB;

68

69 /\* 空闲任务任务栈 \*/

70 static StackType_t Idle_Task_Stack[configMINIMAL_STACK_SIZE];

71 /\* 定时器任务栈 \*/

72 static StackType_t Timer_Task_Stack[configTIMER_TASK_STACK_DEPTH];

73

74 /\* 空闲任务控制块 \*/

75 static StaticTask_t Idle_Task_TCB;

76 /\* 定时器任务控制块 \*/

77 static StaticTask_t Timer_Task_TCB;

78

79 /\*

80 \\*

81 \* 函数声明

82 \\*

83 \*/

84 static void AppTaskCreate(void);/\* 用于创建任务 \*/

85

86 static void LED_Task(void\* pvParameters);/\* LED_Task任务实现 \*/

87

88 static void BSP_Init(void);/\* 用于初始化板载相关资源 \*/

89

90 /*\*

91 \* 使用了静态分配内存，以下这两个函数是由用户实现，函数在task.c文件中有引用

92 \*当且仅当 configSUPPORT_STATIC_ALLOCATION 这个宏定义为 1 的时候才有效

93 \*/

94 void vApplicationGetTimerTaskMemory(StaticTask_t \**ppxTimerTaskTCBBuffer,

95 StackType_t \**ppxTimerTaskStackBuffer,

96 uint32_t \*pulTimerTaskStackSize);

97

98 void vApplicationGetIdleTaskMemory(StaticTask_t \**ppxIdleTaskTCBBuffer,

99 StackType_t \**ppxIdleTaskStackBuffer,

100 uint32_t \*pulIdleTaskStackSize);

101

102 /\*

103 \* @brief 主函数

104 \* @param 无

105 \* @retval 无

106 \* @note 第一步：开发板硬件初始化

107 第二步：创建APP应用任务

108 第三步：启动FreeRTOS，开始多任务调度

109 \/

110 int main(void)

111 {

112 /\* 开发板硬件初始化 \*/

113 BSP_Init();

114 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS-静态创建任务!\r\n");

115 /\* 创建 AppTaskCreate 任务 \*/

116 AppTaskCreate_Handle = xTaskCreateStatic((TaskFunction_t )AppTaskCreate,

117 (const char\* )"AppTaskCreate",//任务名称

118 (uint32_t )128, //任务栈大小

119 (void\* )NULL,//传递给任务函数的参数

120 (UBaseType_t )3, //任务优先级

121 (StackType_t\* )AppTaskCreate_Stack,

122 (StaticTask_t\* )&AppTaskCreate_TCB);

123

124 if (NULL != AppTaskCreate_Handle) /\* 创建成功 \*/

125 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

126

127 while (1); /\* 正常不会执行到这里 \*/

128 }

129

130

131 /\*

132 \* @ 函数名： AppTaskCreate

133 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

134 \* @ 参数：无

135 \* @ 返回值：无

136 \/

137 static void AppTaskCreate(void)

138 {

139 taskENTER_CRITICAL(); //进入临界区

140

141 /\* 创建LED_Task任务 \*/

142 LED_Task_Handle = xTaskCreateStatic((TaskFunction_t )LED_Task,//任务函数

143 (const char*)"LED_Task",//任务名称

144 (uint32_t)128,//任务栈大小

145 (void\* )NULL,//传递给任务函数的参数

146 (UBaseType_t)4,//任务优先级

147 (StackType_t*)LED_Task_Stack,//任务栈

148 (StaticTask_t*)&LED_Task_TCB);//任务控制块

149

150 if (NULL != LED_Task_Handle) /\* 创建成功 \*/

151 printf("LED_Task任务创建成功!\n");

152 else

153 printf("LED_Task任务创建失败!\n");

154

155 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

156

157 taskEXIT_CRITICAL(); //退出临界区

158 }

159

160

161

162 /\*

163 \* @ 函数名： LED_Task

164 \* @ 功能说明： LED_Task任务主体

165 \* @ 参数：

166 \* @ 返回值：无

167 \/

168 static void LED_Task(void\* parameter)

169 {

170 while (1) {

171 LED1_ON;

172 vTaskDelay(500); /\* 延时500个tick \*/

173 printf("led1_task running,LED1_ON\r\n");

174

175 LED1_OFF;

176 vTaskDelay(500); /\* 延时500个tick \*/

177 printf("led1_task running,LED1_OFF\r\n");

178 }

179 }

180

181 /\*

182 \* @ 函数名： BSP_Init

183 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

184 \* @ 参数：

185 \* @ 返回值：无

186 \/

187 static void BSP_Init(void)

188 {

189 /\*

190 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

191 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

192 \* 都统一用这个优先级分组，千万不要再分组，切忌。

193 \*/

194 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

195

196 /\* LED 初始化 \*/

197 LED_GPIO_Config();

198

199 /\* 串口初始化 \*/

200 USART_Config();

201

202 }

203

204

205 /*\*

206 \\*

207 \* @brief 获取空闲任务的任务栈和任务控制块内存

208 \*ppxTimerTaskTCBBuffer : 任务控制块内存

209 \*ppxTimerTaskStackBuffer : 任务栈内存

210 \*pulTimerTaskStackSize : 任务栈大小

211 \* @author fire

212 \* @version V1.0

213 \* @date 2018-xx-xx

214 \\*

215 \*/

216 void vApplicationGetIdleTaskMemory(StaticTask_t \**ppxIdleTaskTCBBuffer,

217 StackType_t \**ppxIdleTaskStackBuffer,

218 uint32_t \*pulIdleTaskStackSize)

219 {

220 \*ppxIdleTaskTCBBuffer=&Idle_Task_TCB;/\* 任务控制块内存 \*/

221 \*ppxIdleTaskStackBuffer=Idle_Task_Stack;/\* 任务栈内存 \*/

222 \*pulIdleTaskStackSize=configMINIMAL_STACK_SIZE;/\* 任务栈大小 \*/

223 }

224

225 /*\*

226 \\*

227 \* @brief 获取定时器任务的任务栈和任务控制块内存

228 \*ppxTimerTaskTCBBuffer : 任务控制块内存

229 \*ppxTimerTaskStackBuffer : 任务栈内存

230 \*pulTimerTaskStackSize : 任务栈大小

231 \* @author fire

232 \* @version V1.0

233 \* @date 2018-xx-xx

234 \\*

235 \*/

236 void vApplicationGetTimerTaskMemory(StaticTask_t \**ppxTimerTaskTCBBuffer,

237 StackType_t \**ppxTimerTaskStackBuffer,

238 uint32_t \*pulTimerTaskStackSize)

239 {

240 \*ppxTimerTaskTCBBuffer=&Timer_Task_TCB;/\* 任务控制块内存 \*/

241 \*ppxTimerTaskStackBuffer=Timer_Task_Stack;/\* 任务栈内存 \*/

242 \*pulTimerTaskStackSize=configTIMER_TASK_STACK_DEPTH;/\* 任务栈大小 \*/

243 }

244

245 /END OF FILE/

246

注意：在使用静态创建任务的时候必须将FreeRTOSConfig.h中的configSUPPORT_STATIC_ALLOCATION宏配置为1。

下载验证
~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的LED灯已经在闪烁，说明我们创建的单任务（使用静态内存）已经跑起来了。

在当前这个例程，任务的栈，任务的控制块用的都是静态内存，必须由用户预先定义，这种方法我们在使用FreeRTOS的时候用的比较少，通常的方法是在任务创建的时候动态的分配任务栈和任务控制块的内存空间，接下来我们讲解下“创建单任务—SRAM动态内存”的方法。

创建单任务—SRAM动态内存
~~~~~~~~~~~~~~

这里，我们创建一个单任务，任务使用的栈和任务控制块是在创建任务的时候FreeRTOS动态分配的，并不是预先定义好的全局变量。那这些动态的内存堆是从哪里来？继续往下看。

动态内存空间的堆从哪里来
^^^^^^^^^^^^

在创建单任务—SRAM静态内存的例程中，任务控制块和任务栈的内存空间都是从内部的SRAM里面分配的，具体分配到哪个地址由编译器决定。现在我们开始使用动态内存，即堆，其实堆也是内存，也属于SRAM。FreeRTOS做法是在SRAM里面定义一个大数组，也就是堆内存，供FreeRTOS的动态内存分配函数使
用，在第一次使用的时候，系统会将定义的堆内存进行初始化，这些代码在FreeRTOS提供的内存管理方案中实现（heap_1.c、heap_2.c、heap_4.c等，具体的内存管理方案后面详细讲解），具体见代码清单12‑11。

代码清单12‑11定义FreeRTOS的堆到内部SRAM

1 //系统所有总的堆大小

2 #define configTOTAL_HEAP_SIZE ((size_t)(36*1024)) **(1)**

3 static uint8_t ucHeap[ configTOTAL_HEAP_SIZE ]; **(2)**

4 /\* 如果这是第一次调用malloc那么堆将需要

5 初始化，以设置空闲块列表。*/

6 if ( pxEnd == NULL )

7 {

8 prvHeapInit(); **(3)**

9 } else

10 {

11 mtCOVERAGE_TEST_MARKER();

12 }

代码清单12‑11 **(1)**\ ：堆内存的大小为configTOTAL_HEAP_SIZE，在FreeRTOSConfig.h中由我们自己定义，configSUPPORT_DYNAMIC_ALLOCATION 这个宏定义在使用FreeRTOS操作系统的时候必须开启。

代码清单12‑11\ **(2)**\ ：从内部SRAMM里面定义一个静态数组ucHeap，大小由configTOTAL_HEAP_SIZE这个宏决定，目前定义为36KB。定义的堆大小不能超过内部SRAM的总大小。

代码清单12‑11\ **(3)**\ ：如果这是第一次调用malloc那么需要将堆进行初始化，以设置空闲块列表，方便以后分配内存，初始化完成之后会取得堆的结束地址，在MemMang中的5个内存分配heap_x.c文件中实现。

.. _定义任务函数-1:

定义任务函数
^^^^^^

使用动态内存的时候，任务的主体函数与使用静态内存时是一样的，具体见代码清单12‑12。

代码清单12‑12定义任务函数

1 static voidLED_Task (void\* parameter)

2 {

3 while (1) **(1)**

4 {

5 LED1_ON;

6 vTaskDelay(500); /\* 延时500个tick \*/**(2)**

7

8 LED1_OFF;

9 vTaskDelay(500); /\* 延时500个tick \*/

10

11 }

12 }

代码清单12‑12\ **(1)**\ ：任务必须是一个死循环，否则任务将通过LR返回，如果LR指向了非法的内存就会产生HardFault_Handler，而FreeRTOS指向一个任务退出函数prvTaskExitError()，里面是一个死循环，那么任务返回之后就在死循环中执行，这样子的任务是不
安全的，所以避免这种情况，任务一般都是死循环并且无返回值的。我们的AppTaskCreate任务，执行一次之后就进行删除，则不影响系统运行，所以，只执行一次的任务在执行完毕要记得及时删除。

代码清单12‑12\ **(2)**\ ：任务里面的延时函数必须使用FreeRTOS里面提供的延时函数，并不能使用我们裸机编程中的那种延时。这两种的延时的区别是FreeRTOS里面的延时是阻塞延时，即调用vTaskDelay()函数的时候，当前任务会被挂起，调度器会切换到其他就绪的任务，从而实现多任
务。如果还是使用裸机编程中的那种延时，那么整个任务就成为了一个死循环，如果恰好该任务的优先级是最高的，那么系统永远都是在这个任务中运行，比它优先级更低的任务无法运行，根本无法实现多任务。

.. _定义任务栈-1:

定义任务栈
^^^^^

使用动态内存的时候，任务栈在任务创建的时候创建，不用跟使用静态内存那样要预先定义好一个全局的静态的栈空间，动态内存就是按需分配内存，随用随取。

定义任务控制块指针
^^^^^^^^^

使用动态内存时候，不用跟使用静态内存那样要预先定义好一个全局的静态的任务控制块空间。任务控制块是在任务创建的时候分配内存空间创建，任务创建函数会返回一个指针，用于指向任务控制块，所以要预先为任务栈定义一个任务控制块指针，也是我们常说的任务句柄，具体见代码清单12‑13。

代码清单12‑13定义任务句柄

1 /\* 任务句柄 \/

2 /\*

3 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

4 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

5 \* 这个句柄可以为NULL。

6 \*/

7 /\* 创建任务句柄 \*/

8 static TaskHandle_t AppTaskCreate_Handle = NULL;

9 /\* LED任务句柄 \*/

10 static TaskHandle_t LED_Task_Handle = NULL;

动态创建任务
^^^^^^

使用静态内存时，使用xTaskCreateStatic()来创建一个任务，而使用动态内存的时，则使用xTaskCreate()函数来创建一个任务，两者的函数名不一样，具体的形参也有区别，具体见代码清单12‑14。

代码清单12‑14动态创建任务

1 /\* 创建AppTaskCreate任务 \*/

2 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate, /\* 任务入口函数 \*/**(1)**

3 (const char\* )"AppTaskCreate",/\* 任务名字 \*/**(2)**

4 (uint16_t )512, /\* 任务栈大小 \*/ **(3)**

5 (void\* )NULL,/\* 任务入口函数参数 \*/ **(4)**

6 (UBaseType_t )1, /\* 任务的优先级 \*/ **(5)**

7 (TaskHandle_t\* )&AppTaskCreate_Handle);/\* 任务控制块指针 \*/**(6)**

8 /\* 启动任务调度 \*/

9 if (pdPASS == xReturn)

10 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

代码清单12‑14\ **(1)**\ ：任务入口函数，即任务函数的名称，需要我们自己定义并且实现。

代码清单12‑14\ **(2)**\ ：任务名字，字符串形式，最大长度由FreeRTOSConfig.h中定义的configMAX_TASK_NAME_LEN宏指定，多余部分会被自动截掉，这里任务名字最好要与任务函数入口名字一致，方便进行调试。

代码清单12‑14\ **(3)**\ ：任务栈大小，单位为字，在32位的处理器下（STM32），一个字等于4个字节，那么任务大小就为128 \* 4字节。

代码清单12‑14\ **(4)**\ ：任务入口函数形参，不用的时候配置为0或者NULL即可。

代码清单12‑14\ **(5)**\ ：任务的优先级。优先级范围根据FreeRTOSConfig.h中的宏configMAX_PRIORITIES决定，如果使能configUSE_PORT_OPTIMISED_TASK_SELECTION，这个宏定义，则最多支持32个优先级；如果不用特殊方法查找下
一个运行的任务，那么则不强制要求限制最大可用优先级数目。在FreeRTOS中，数值越大优先级越高，0代表最低优先级。

代码清单12‑14\ **(6)**\
：任务控制块指针，在使用内存的时候，需要给任务初始化函数xTaskCreateStatic()传递预先定义好的任务控制块的指针。在使用动态内存的时候，任务创建函数xTaskCreate()会返回一个指针指向任务控制块，该任务控制块是xTaskCreate()函数里面动态分配的一块内存。

.. _启动任务-1:

启动任务
^^^^

当任务创建好后，是处于任务就绪（Ready），在就绪态的任务可以参与操作系统的调度。但是此时任务仅仅是创建了，还未开启任务调度器，也没创建空闲任务与定时器任务（如果使能了configUSE_TIMERS这个宏定义），那这两个任务就是在启动任务调度器中实现，每个操作系统，任务调度器只启动一次，之后就不
会再次执行了，FreeRTOS中启动任务调度器的函数是vTaskStartScheduler()，并且启动任务调度器的时候就不会返回，从此任务管理都由FreeRTOS管理，此时才是真正进入实时操作系统中的第一步，具体见代码清单12‑15。

代码清单12‑15启动任务

1 /\* 启动任务调度 \*/

2 if (pdPASS == xReturn)

3 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

4 else

5 return -1;

.. _main.c文件内容全貌-1:

main.c文件内容全貌
^^^^^^^^^^^^

与代码清单12‑10中创建单任务的思路一致，我们统一在AppTaskCreate中创建其它用户任务，并且把任务主体，任务栈，任务控制块这三部分代码统一放到main.c中。

现在我们把任务主体，任务栈，任务控制块这三部分代码统一放到main.c中，我们在main.c文件中创建一个AppTaskCreate任务，这个任务是用于创建用户任务，为了方便管理，我们的所有的任务创建都统一放在这个函数中，在这个函数中创建成功的任务就可以直接参与任务调度了，具体内容见代码清单12‑1
6。

代码清单12‑16main.c文件内容全貌

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS v9.0.0 + STM32 工程模版

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

26 /\* 开发板硬件bsp头文件 \*/

27 #include"bsp_led.h"

28 #include"bsp_usart.h"

29

30 /\* 任务句柄 \/

31 /\*

32 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

33 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

34 \* 这个句柄可以为NULL。

35 \*/

36 /\* 创建任务句柄 \*/

37 static TaskHandle_t AppTaskCreate_Handle = NULL;

38 /\* LED任务句柄 \*/

39 static TaskHandle_t LED_Task_Handle = NULL;

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

61 /\*

62 \\*

63 \* 函数声明

64 \\*

65 \*/

66 static void AppTaskCreate(void);/\* 用于创建任务 \*/

67

68 static void LED_Task(void\* pvParameters);/\* LED_Task任务实现 \*/

69

70 static void BSP_Init(void);/\* 用于初始化板载相关资源 \*/

71

72 /\*

73 \* @brief 主函数

74 \* @param 无

75 \* @retval 无

76 \* @note 第一步：开发板硬件初始化

77 第二步：创建APP应用任务

78 第三步：启动FreeRTOS，开始多任务调度

79 \/

80 int main(void)

81 {

82 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

83

84 /\* 开发板硬件初始化 \*/

85 BSP_Init();

86 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS-工程模板!\r\n");

87 /\* 创建AppTaskCreate任务 \*/

88 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate, /\* 任务入口函数 \*/

89 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

90 (uint16_t )512, /\* 任务栈大小 \*/

91 (void\* )NULL,/\* 任务入口函数参数 \*/

92 (UBaseType_t )1, /\* 任务的优先级 \*/

93 (TaskHandle_t\* )&AppTaskCreate_Handle);/\* 任务控制块指针 \*/

94 /\* 启动任务调度 \*/

95 if (pdPASS == xReturn)

96 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

97 else

98 return -1;

99

100 while (1); /\* 正常不会执行到这里 \*/

101 }

102

103

104 /\*

105 \* @ 函数名： AppTaskCreate

106 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

107 \* @ 参数：无

108 \* @ 返回值：无

109 \/

110 static void AppTaskCreate(void)

111 {

112 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

113

114 taskENTER_CRITICAL(); //进入临界区

115

116 /\* 创建LED_Task任务 \*/

117 xReturn = xTaskCreate((TaskFunction_t )LED_Task, /\* 任务入口函数 \*/

118 (const char\* )"LED_Task",/\* 任务名字 \*/

119 (uint16_t )512, /\* 任务栈大小 \*/

120 (void\* )NULL, /\* 任务入口函数参数 \*/

121 (UBaseType_t )2, /\* 任务的优先级 \*/

122 (TaskHandle_t\* )&LED_Task_Handle);/\* 任务控制块指针 \*/

123 if (pdPASS == xReturn)

124 printf("创建LED_Task任务成功!\r\n");

125

126 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

127

128 taskEXIT_CRITICAL(); //退出临界区

129 }

130

131

132

133 /\*

134 \* @ 函数名： LED_Task

135 \* @ 功能说明： LED_Task任务主体

136 \* @ 参数：

137 \* @ 返回值：无

138 \/

139 static void LED_Task(void\* parameter)

140 {

141 while (1) {

142 LED1_ON;

143 vTaskDelay(500); /\* 延时500个tick \*/

144 printf("led1_task running,LED1_ON\r\n");

145

146 LED1_OFF;

147 vTaskDelay(500); /\* 延时500个tick \*/

148 printf("led1_task running,LED1_OFF\r\n");

149 }

150 }

151

152 /\*

153 \* @ 函数名： BSP_Init

154 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

155 \* @ 参数：

156 \* @ 返回值：无

157 \/

158 static void BSP_Init(void)

159 {

160 /\*

161 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

162 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

163 \* 都统一用这个优先级分组，千万不要再分组，切忌。

164 \*/

165 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

166

167 /\* LED 初始化 \*/

168 LED_GPIO_Config();

169

170 /\* 串口初始化 \*/

171 USART_Config();

172

173 }

174

175 /END OF FILE/

176

其实动态创建与静态创建的差别就是特别小，以后我们使用FreeRTOS除非是特别说明，否则我们都使用动态创建任务。

.. _下载验证-1:

下载验证
~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的LED灯已经在闪烁，说明我们创建的单任务（使用动态内存）已经跑起来了。在往后的实验中，我们创建内核对象均采用动态内存分配方案。

创建多任务—SRAM动态内存
~~~~~~~~~~~~~~

创建多任务只需要按照创建单任务的套路依葫芦画瓢即可，接下来我们创建两个任务，任务1让一个LED灯闪烁，任务2让另外一个LED闪烁，两个LED闪烁的频率不一样，具体实现见代码清单12‑17的加粗部分，两个任务的优先级不一样。

代码清单12‑17创建多任务—SRAM动态内存

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS v9.0.0 + STM32 多任务创建

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

26 /\* 开发板硬件bsp头文件 \*/

27 #include"bsp_led.h"

28 #include"bsp_usart.h"

29

30 /\* 任务句柄 \/

31 /\*

32 \* 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄

33 \* 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么

34 \* 这个句柄可以为NULL。

35 \*/

36 /\* 创建任务句柄 \*/

37 static TaskHandle_t AppTaskCreate_Handle = NULL;

**38 /\* LED1任务句柄 \*/**

**39 static TaskHandle_t LED1_Task_Handle = NULL;**

**40 /\* LED2任务句柄 \*/**

**41 static TaskHandle_t LED2_Task_Handle = NULL;**

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

54

55

56 /\* 全局变量声明 \/

57 /\*

58 \* 当我们在写应用程序的时候，可能需要用到一些全局变量。

59 \*/

60

61

62 /\*

63 \\*

64 \* 函数声明

65 \\*

66 \*/

67 static void AppTaskCreate(void);/\* 用于创建任务 \*/

68

**69 static void LED1_Task(void\* pvParameters);/\* LED1_Task任务实现 \*/**

**70 static void LED2_Task(void\* pvParameters);/\* LED2_Task任务实现 \*/**

71

72 static void BSP_Init(void);/\* 用于初始化板载相关资源 \*/

73

74 /\*

75 \* @brief 主函数

76 \* @param 无

77 \* @retval 无

78 \* @note 第一步：开发板硬件初始化

79 第二步：创建APP应用任务

80 第三步：启动FreeRTOS，开始多任务调度

81 \/

82 int main(void)

83 {

84 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

85

86 /\* 开发板硬件初始化 \*/

87 BSP_Init();

88 printf("这是一个[野火]-STM32全系列开发板-FreeRTOS-多任务创建实验!\r\n");

89 /\* 创建AppTaskCreate任务 \*/

90 xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate, /\* 任务入口函数 \*/

91 (const char\* )"AppTaskCreate",/\* 任务名字 \*/

92 (uint16_t )512, /\* 任务栈大小 \*/

93 (void\* )NULL,/\* 任务入口函数参数 \*/

94 (UBaseType_t )1, /\* 任务的优先级 \*/

95 (TaskHandle_t\* )&AppTaskCreate_Handle);/\* 任务控制块指针 \*/

96 /\* 启动任务调度 \*/

97 if (pdPASS == xReturn)

98 vTaskStartScheduler(); /\* 启动任务，开启调度 \*/

99 else

100 return -1;

101

102 while (1); /\* 正常不会执行到这里 \*/

103 }

104

105

106 /\*

107 \* @ 函数名： AppTaskCreate

108 \* @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面

109 \* @ 参数：无

110 \* @ 返回值：无

111 \/

112 static void AppTaskCreate(void)

113 {

114 BaseType_t xReturn = pdPASS;/\* 定义一个创建信息返回值，默认为pdPASS \*/

115

116 taskENTER_CRITICAL(); //进入临界区

117

**118 /\* 创建LED_Task任务 \*/**

**119 xReturn = xTaskCreate((TaskFunction_t )LED1_Task, /\* 任务入口函数 \*/**

**120 (const char\* )"LED1_Task",/\* 任务名字 \*/**

**121 (uint16_t )512, /\* 任务栈大小 \*/**

**122 (void\* )NULL,/\* 任务入口函数参数 \*/**

**123 (UBaseType_t )2, /\* 任务的优先级 \*/**

**124 (TaskHandle_t\* )&LED1_Task_Handle);/\* 任务控制块指针 \*/**

**125 if (pdPASS == xReturn)**

**126 printf("创建LED1_Task任务成功!\r\n");**

**127**

**128 /\* 创建LED_Task任务 \*/**

**129 xReturn = xTaskCreate((TaskFunction_t )LED2_Task, /\* 任务入口函数 \*/**

**130 (const char\* )"LED2_Task",/\* 任务名字 \*/**

**131 (uint16_t )512, /\* 任务栈大小 \*/**

**132 (void\* )NULL,/\* 任务入口函数参数 \*/**

**133 (UBaseType_t )3, /\* 任务的优先级 \*/**

**134 (TaskHandle_t\* )&LED2_Task_Handle);/\* 任务控制块指针 \*/**

**135 if (pdPASS == xReturn)**

**136 printf("创建LED2_Task任务成功!\r\n");**

137

138 vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

139

140 taskEXIT_CRITICAL(); //退出临界区

141 }

142

143

144

145 /\*

146 \* @ 函数名： LED1_Task

147 \* @ 功能说明： LED1_Task任务主体

148 \* @ 参数：

149 \* @ 返回值：无

150 \/

**151 static void LED1_Task(void\* parameter)**

**152 {**

**153 while (1) {**

**154 LED1_ON;**

**155 vTaskDelay(500); /\* 延时500个tick \*/**

**156 printf("led1_task running,LED1_ON\r\n");**

**157**

**158 LED1_OFF;**

**159 vTaskDelay(500); /\* 延时500个tick \*/**

**160 printf("led1_task running,LED1_OFF\r\n");**

**161 }**

**162 }**

163

164 /\*

165 \* @ 函数名： LED2_Task

166 \* @ 功能说明： LED2_Task任务主体

167 \* @ 参数：

168 \* @ 返回值：无

169 \/

**170 static void LED2_Task(void\* parameter)**

**171 {**

**172 while (1) {**

**173 LED2_ON;**

**174 vTaskDelay(1000); /\* 延时500个tick \*/**

**175 printf("led1_task running,LED2_ON\r\n");**

**176**

**177 LED2_OFF;**

**178 vTaskDelay(1000); /\* 延时500个tick \*/**

**179 printf("led1_task running,LED2_OFF\r\n");**

**180 }**

**181 }**

182 /\*

183 \* @ 函数名： BSP_Init

184 \* @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面

185 \* @ 参数：

186 \* @ 返回值：无

187 \/

188 static void BSP_Init(void)

189 {

190 /\*

191 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

192 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

193 \* 都统一用这个优先级分组，千万不要再分组，切忌。

194 \*/

195 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

196

197 /\* LED 初始化 \*/

198 LED_GPIO_Config();

199

200 /\* 串口初始化*/

201 USART_Config();

202

203 }

204

205 /END OF FILE/

206

目前多任务我们只创建了两个，如果要创建3个、4个甚至更多都是同样的套路，容易忽略的地方是任务栈的大小，每个任务的优先级。大的任务，栈空间要设置大一点，重要的任务优先级要设置的高一点。

.. _下载验证-2:

下载验证
~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的两个LED灯以不同的频率在闪烁，说明我们创建的多任务（使用动态内存）已经跑起来了。在往后的实验中，我们创建内核对象均采用动态内存分配方案。
