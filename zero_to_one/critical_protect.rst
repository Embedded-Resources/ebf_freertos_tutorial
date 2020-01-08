.. vim: syntax=rst

临界段的保护
------

什么是临界段
~~~~~~

临界段用一句话概括就是一段在执行的时候不能被中断的代码段。在FreeRTOS里面，这个临界段最常出现的就是对全局变量的操作，全局变量就好像是一个枪把子，谁都可以对他开枪，但是我开枪的时候，你就不能开枪，否则就不知道是谁命中了靶子。可能有人会说我可以在子弹上面做个标记，我说你能不能不要瞎扯淡。

那么什么情况下临界段会被打断？一个是系统调度，还有一个就是外部中断。在FreeRTOS，系统调度，最终也是产生PendSV中断，在PendSV Handler里面实现任务的切换，所以还是可以归结为中断。既然这样，FreeRTOS对临界段的保护最终还是回到对中断的开和关的控制。

Cortex-M内核快速关中断指令
~~~~~~~~~~~~~~~~~

为了快速地开关中断， Cortex-M内核专门设置了一条 CPS 指令，有 4 种用法，具体见代码清单6‑1。

代码清单6‑1 CPS 指令用法

1 CPSID I ;PRIMASK=1 ;关中断

2 CPSIE I ;PRIMASK=0 ;开中断

3 CPSID F ;FAULTMASK=1 ;关异常

4 CPSIE F ;FAULTMASK=0 ;开异常

代码清单6‑1中PRIMASK和FAULTMAST是Cortex-M内核里面三个中断屏蔽寄存器中的两个，还有一个是BASEPRI，有关这三个寄存器的详细用法见表6‑1。

表6‑1Cortex-M内核中断屏蔽寄存器组描述

.. list-table::
   :widths: 50 50
   :header-rows: 0


   * - 名字      |
     - 能描述                                                |

   * - PRIMASK
     - 这是个只有单一比特的寄存器。在它被置1                   | 后，就关掉所有可屏蔽的异常，只剩下NMI                   | 和硬FAULT可以响应。它的缺省值是0，表示没有关中断。      |

   * - FAULTMASK
     - 这是个只有1 个位的寄存器。当它置1 时，只有NMI           | 才能响应，所有其他的异常，甚至是                        | 硬FAULT，也通通闭嘴。它的缺省值也是0，表示没有关异常。  |

   * - BASEPRI
     - 这个寄存器最多有9                                       | 位（由表达优先                                          | 级的位数决定）。它定义了被屏蔽优先级的阈值。当它被设成  |
       某个值后，所有优先级号大于等于此值的中断都被关（优先级  | 号越大，优先级越低）。但若被设成0，则不关闭任何中断，0  | 也是缺省值。                                            |


但是，在FreeRTOS中，对中断的开和关是通过操作BASEPRI寄存器来实现的，即大于等于BASEPRI的值的中断会被屏蔽，小于BASEPRI的值的中断则不会被屏蔽，不受FreeRTOS管理。用户可以设置BASEPRI的值来选择性的给一些非常紧急的中断留一条后路。

关中断
~~~

FreeRTOS关中断的函数在portmacro.h中定义，分不带返回值和带返回值两种，具体实现见代码清单6‑2。

代码清单6‑2关中断函数

1 /\* 不带返回值的关中断函数，不能嵌套，不能在中断里面使用 \*/**(1)**

2 #define portDISABLE_INTERRUPTS() vPortRaiseBASEPRI()

3

4 void vPortRaiseBASEPRI( void )

5 {

6 uint32_t ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;\ **(1)-①**

7 \__asm

8 {

9 msr basepri, ulNewBASEPRI\ **(1)-②**

10 dsb

11 isb

12 }

13 }

14

15 /\* 带返回值的关中断函数，可以嵌套，可以在中断里面使用 \*/**(2)**

16 #define portSET_INTERRUPT_MASK_FROM_ISR() ulPortRaiseBASEPRI()

17 ulPortRaiseBASEPRI( void )

18 {

19 uint32_t ulReturn, ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;\ **(2)-①**

20 \__asm

21 {

22 mrs ulReturn, basepri\ **(2)-②**

23 msr basepri, ulNewBASEPRI\ **(2)-③**

24 dsb

25 isb

26 }

27 return ulReturn;\ **(2)-④**

28 }

不带返回值的关中断函数
^^^^^^^^^^^

代码清单6‑2\ **(1)**\ ：不带返回值的关中断函数，不能嵌套，不能在中断里面使用。不带返回值的意思是：在往BASEPRI写入新的值的时候，不用先将BASEPRI的值保存起来，即不用管当前的中断状态是怎么样的，既然不用管当前的中断状态，也就意味着这样的函数不能在中断里面调用。

代码清单6‑2\ **(1)-①**\ ：configMAX_SYSCALL_INTERRUPT_PRIORITY是一个在FreeRTOSConfig.h中定义的宏，即要写入到BASEPRI寄存器的值。该宏默认定义为191，高四位有效，即等于0xb0，或者是11，即优先级大于等于11的中断都会被屏蔽
，11以内的中断则不受FreeRTOS管理。

代码清单6‑2\ **(1)-②**\ ：将configMAX_SYSCALL_INTERRUPT_PRIORITY的值写入BASEPRI寄存器，实现关中断（准确来说是关部分中断）。

带返回值的关中断函数
^^^^^^^^^^

代码清单6‑2\ **(2)**\ ：带返回值的关中断函数，可以嵌套，可以在中断里面使用。带返回值的意思是：在往BASEPRI写入新的值的时候，先将BASEPRI的值保存起来，在更新完BASEPRI的值的时候，将之前保存好的BASEPRI的值返回，返回的值作为形参传入开中断函数。

代码清单6‑2\ **(2)-①**\ ：configMAX_SYSCALL_INTERRUPT_PRIORITY是一个在FreeRTOSConfig.h中定义的宏，即要写入到BASEPRI寄存器的值。该宏默认定义为191，高四位有效，即等于0xb0，或者是11，即优先级大于等于11的中断都会被屏蔽
，11以内的中断则不受FreeRTOS管理

代码清单6‑2\ **(2)-②**\ ：保存BASEPRI的值，记录当前哪些中断被关闭。

代码清单6‑2\ **(2)-③**\ ：更新BASEPRI的值。

代码清单6‑2\ **(2)-④**\ ：返回原来BASEPRI的值。

开中断
~~~

FreeRTOS开中断的函数在portmacro.h中定义，具体实现见代码清单6‑3。

代码清单6‑3开中断函数

1 /\* 不带中断保护的开中断函数 \*/

2 #define portENABLE_INTERRUPTS() vPortSetBASEPRI( 0 )\ **(2)**

3

4 /\* 带中断保护的开中断函数 \*/

5 #define portCLEAR_INTERRUPT_MASK_FROM_ISR(x) vPortSetBASEPRI(x)\ **(3)**

6

7 void vPortSetBASEPRI( uint32_t ulBASEPRI )\ **(1)**

8 {

9 \__asm

10 {

11 msr basepri, ulBASEPRI

12 }

13 }

代码清单6‑3\ **(1)**\ ：开中断函数，具体是将传进来的形参更新到BASEPRI寄存器。根据传进来形参的不同，分为中断保护版本与非中断保护版本。

代码清单6‑3\ **(2)**\ ：不带中断保护的开中断函数，直接将BASEPRI的值设置为0，与portDISABLE_INTERRUPTS()成对使用。

代码清单6‑3\ **(3)**\ ：带中断保护的开中断函数，将上一次关中断时保存的BASEPRI的值作为形参，与portSET_INTERRUPT_MASK_FROM_ISR()成对使用。

进入/退出临界段的宏
~~~~~~~~~~

进入和退出临界段的宏在task.h中定义，具体见代码清单6‑4。

代码清单6‑4进入和退出临界段宏定义

1 #define taskENTER_CRITICAL() portENTER_CRITICAL()

2 #define taskENTER_CRITICAL_FROM_ISR() portSET_INTERRUPT_MASK_FROM_ISR()

3

4 #define taskEXIT_CRITICAL() portEXIT_CRITICAL()

5 #define taskEXIT_CRITICAL_FROM_ISR( x ) portCLEAR_INTERRUPT_MASK_FROM_ISR( x )

进入和退出临界段的宏分中断保护版本和非中断版本，但最终都是通过开/关中断来实现。有关开/光中断的底层代码我们已经讲解，那么接下来的退出和进入临界段的代码配套注释来理解即可。

进入临界段
^^^^^

进入临界段，不带中断保护版本且不能嵌套的代码实现具体见代码清单6‑5。

不带中断保护版本，不能嵌套
'''''''''''''

代码清单6‑5进入临界段，不带中断保护版本，不能嵌套

1 /\* ==========进入临界段，不带中断保护版本，不能嵌套=============== \*/

2 /\* 在task.h中定义 \*/

3 #define taskENTER_CRITICAL() portENTER_CRITICAL()

4

5 /\* 在portmacro.h中定义 \*/

6 #define portENTER_CRITICAL() vPortEnterCritical()

7

8 /\* 在port.c中定义 \*/

9 void vPortEnterCritical( void )

10 {

11 portDISABLE_INTERRUPTS();

12 uxCriticalNesting++;\ **(1)**

13

14 if ( uxCriticalNesting == 1 )\ **(2)**

15 {

16 configASSERT( ( portNVIC_INT_CTRL_REG & portVECTACTIVE_MASK ) == 0 );

17 }

18 }

19

20 /\* 在portmacro.h中定义 \*/

21 #define portDISABLE_INTERRUPTS() vPortRaiseBASEPRI()

22

23 /\* 在portmacro.h中定义 \*/

24 static portFORCE_INLINE void vPortRaiseBASEPRI( void )

25 {

26 uint32_t ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;

27

28 \__asm

29 {

30 msr basepri, ulNewBASEPRI

31 dsb

32 isb

33 }

34 }

代码清单6‑5\ **(1)**\
：uxCriticalNesting是在port.c中定义的静态变量，表示临界段嵌套计数器，默认初始化为0xaaaaaaaa，在调度器启动时会被重新初始化为0：vTaskStartScheduler()->xPortStartScheduler()->uxCriticalNesting = 0。

代码清单6‑5\ **(2)**\ ：如果uxCriticalNesting等于1，即一层嵌套，要确保当前没有中断活跃，即内核外设SCB中的中断和控制寄存器SCB_ICSR的低8位要等于0。有关SCB_ICSR的具体描述可参考“STM32F10xxx Cortex-M3 programming
manual-4.4.2小节”。

进入临界段，带中断保护版本且可以嵌套的代码实现具体见代码清单6‑6。

带中断保护版本，可以嵌套
''''''''''''

代码清单6‑6进入临界段，带中断保护版本，可以嵌套

1 /\* ==========进入临界段，带中断保护版本，可以嵌套=============== \*/

2 /\* 在task.h中定义 \*/

3 #define taskENTER_CRITICAL_FROM_ISR() portSET_INTERRUPT_MASK_FROM_ISR()

4

5 /\* 在portmacro.h中定义 \*/

6 #define portSET_INTERRUPT_MASK_FROM_ISR() ulPortRaiseBASEPRI()

7

8 /\* 在portmacro.h中定义 \*/

9 static portFORCE_INLINE uint32_t ulPortRaiseBASEPRI( void )

10 {

11 uint32_t ulReturn, ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;

12

13 \__asm

14 {

15 mrs ulReturn, basepri

16 msr basepri, ulNewBASEPRI

17 dsb

18 isb

19 }

20

21 return ulReturn;

22 }

退出临界段
^^^^^

退出临界段，不带中断保护版本且不能嵌套的代码实现具体见代码清单6‑7。

不带中断保护的版本，不能嵌套
''''''''''''''

代码清单6‑7退出临界段，不带中断保护版本，不能嵌套

1 /\* ==========退出临界段，不带中断保护版本，不能嵌套=============== \*/

2 /\* 在task.h中定义 \*/

3 #define taskEXIT_CRITICAL() portEXIT_CRITICAL()

4

5 /\* 在portmacro.h中定义 \*/

6 #define portEXIT_CRITICAL() vPortExitCritical()

7

8 /\* 在port.c中定义 \*/

9 void vPortExitCritical( void )

10 {

11 configASSERT( uxCriticalNesting );

12 uxCriticalNesting--;

13 if ( uxCriticalNesting == 0 )

14 {

15 portENABLE_INTERRUPTS();

16 }

17 }

18

19 /\* 在portmacro.h中定义 \*/

20 #define portENABLE_INTERRUPTS() vPortSetBASEPRI( 0 )

21

22 /\* 在portmacro.h中定义 \*/

23 static portFORCE_INLINE void vPortSetBASEPRI( uint32_t ulBASEPRI )

24 {

25 \__asm

26 {

27 msr basepri, ulBASEPRI

28 }

29 }

带中断保护的版本，可以嵌套
'''''''''''''

代码清单6‑8退出临界段，带中断保护版本，可以嵌套

1 /\* ==========退出临界段，带中断保护版本，可以嵌套=============== \*/

2 /\* 在task.h中定义 \*/

3 #define taskEXIT_CRITICAL_FROM_ISR( x ) portCLEAR_INTERRUPT_MASK_FROM_ISR( x )

4

5 /\* 在portmacro.h中定义 \*/

6 #define portCLEAR_INTERRUPT_MASK_FROM_ISR(x) vPortSetBASEPRI(x)

7

8 /\* 在portmacro.h中定义 \*/

9 static portFORCE_INLINE void vPortSetBASEPRI( uint32_t ulBASEPRI )

10 {

11 \__asm

12 {

13 msr basepri, ulBASEPRI

14 }

15 }

临界段代码的应用
~~~~~~~~

在FreeRTOS中，对临界段的保护出现在两种场合，一种是在中断场合一种是在非中断场合，具体的应用见。

代码清单6‑9临界段代码应用

1 /\* 在中断场合，临界段可以嵌套 \*/

2 {

3 uint32_t ulReturn;

4 /\* 进入临界段，临界段可以嵌套 \*/

5 ulReturn = taskENTER_CRITICAL_FROM_ISR();

6

7 /\* 临界段代码 \*/

8

9 /\* 退出临界段 \*/

10 taskEXIT_CRITICAL_FROM_ISR( ulReturn );

11 }

12

13 /\* 在非中断场合，临界段不能嵌套 \*/

14 {

15 /\* 进入临界段 \*/

16 taskENTER_CRITICAL();

17

18 /\* 临界段代码 \*/

19

20 /\* 退出临界段*/

21 taskEXIT_CRITICAL();

22 }

实验现象
~~~~

本章没有实验，充分理解本章内容即可，这么简单，其实也没啥好理解的。
