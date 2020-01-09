.. vim: syntax=rst

移植FreeRTOS到STM32
=====================

本章开始，先新建一个基于野火STM32全系列（包含M3/4/7）开发板的的FreeRTOS的工程模板，让FreeRTOS先跑起来。以后所有的FreeRTOS相关的例程我们都在此模板上修改和添加代码，不用再反反复复地新建。在本书配套的例程中，每一章的例程对野火STM32的每一个板子都会有一个对应的例程
，但是区别都很小，如果有区别的地方我会在教程里面详细指出，如果没有特别备注那么都是一样的。

获取STM32的裸机工程模板
~~~~~~~~~~~~~~

STM32的裸机工程模板我们直接使用野火STM32开发板配套的固件库例程即可。这里我们选取比较简单的例程—“GPIO输出—使用固件库点亮LED”作为裸机工程模板。该裸机工程模板均可以在对应板子的A盘/程序源码/固件库例程的目录下获取到，下面以野火F103-霸道板子的光盘目录为例，具体见图11‑1。

|portin002|

图11‑1 STM32裸机工程模板在光盘资料中的位置

下载FreeRTOS V9.0.0源码
~~~~~~~~~~~~~~~~~~~

在移植之前，我们首先要获取到FreeRTOS的官方的源码包。这里我们提供两个下载链接，一个是官网：\ http://www.freertos.org/\ ，另外一个是代码托管网站：\ https://sourceforge.net/projects/freertos/files/FreeRTOS/\
。这里我们演示如何在代码托管网站里面下载。打开网站链接之后，我们选择FreeRTOS的最新版本V9.0.0（2016年），尽管现在FreeRTOS的版本已经更新到V10.0.1了，但是我们还是选择V9.0.0，因为内核很稳定，并且网上资料很多，因为V10.0.0版本之后是亚马逊收购了FreeRTOS
之后才出来的版本，主要添加了一些云端组件，我们本书所讲的FreeRTOS是实时内核，采用V9.0.0版本足以。

我们打开FreeRTOS的代码托管网站，就可以看到FreeRTOS的源码及其版本信息了，具体见图11‑2。

|portin003|

图11‑2FreeRTOS V9.0.0版本源码

点击V9.0.0会跳转到版本目录，具体见图11‑3。这里有zip和exe格式的压缩包，其实都是FreeRTOS的源码，只是压缩的格式不一样，所以大小也不一样，我们这里选择zip的下载，点击了就会出现下载的链接，下载完成解压后就可以得到我们想要的FreeRTOS
V9.0.0版本的源码了，具体见图11‑4。

|portin004|

图11‑3 FreeRTOS源码包下载链接

|portin005|

图11‑4 FreeRTOSv9.0.0源码

FreeRTOS文件夹内容简介
~~~~~~~~~~~~~~~

FreeRTOS文件夹
^^^^^^^^^^^

FreeRTOS包含Demo例程和内核源码（比较重要，我们就需要提取该目录下的大部分文件），具体见图11‑5。FreeRTOS文件夹下的 Source文件夹里面包含的是FreeRTOS内核的源代码，我们移植FreeRTOS的时候就需要这部分源代码；FreeRTOS文件夹下的Demo文件夹里面包含了F
reeRTOS官方为各个单片机移植好的工程代码，FreeRTOS为了推广自己，会给各种半导体厂商的评估板写好完整的工程程序，这些程序就放在Demo这个目录下，这部分Demo非常有参考价值。我们把FreeRTOS到STM32的时候，FreeRTOSConfig.h这个头文件就是从这里拷贝过来的，下面我
们对FreeRTOS的文件夹进行分析说明。

|portin006|

图11‑5FreeRTOS文件夹内容

Source文件夹
'''''''''

这里我们再重点分析下FreeRTOS/ Source文件夹下的文件，具体见图11‑6。编号\ **①**\ 和\ **③**\ 包含的是FreeRTOS的通用的头文件和C文件，这两部分的文件试用于各种编译器和处理器，是通用的。需要移植的头文件和C文件放在编号\ **②**\
portblle这个文件夹。

|portin007|

图11‑6 Source文件夹内容

我们打开portblle这个文件夹，可以看到里面很多与编译器相关的文件夹，在不同的编译器中使用不同的支持文件。编号\ **①**\ 中的KEIL就是我们就是我们使用的编译器，当年打开KEIL文件夹的时候，你会看到一句话“See-also-the-RVDS-
directory.txt”，其实KEIL里面的内容跟RVDS里面的内容一样，所以我们只需要编号\ **③**\ RVDS文件夹里面的内容即可。而编号\ **②**\ MemMang文件夹下存放的是跟内存管理相关的，稍后具体介绍，portblle文件夹内容具体见图11‑7。

|portin008|

图11‑7 portblle文件夹内容

打开RVDS文件夹，下面包含了各种处理器相关的文件夹，从文件夹的名字我们就非常熟悉了，我们学习的STM32有M0、M3、M4等各种系列，FreeRTOS是一个软件，单片机是一个硬件，FreeRTOS要想运行在一个单片机上面，它们就必须关联在一起，那么怎么关联？还是得通过写代码来关联，这部分关联的文件
叫接口文件，通常由汇编和C联合编写。这些接口文件都是跟硬件密切相关的，不同的硬件接口文件是不一样的，但都大同小异。编写这些接口文件的过程我们就叫移植，移植的过程通常由FreeRTOS和mcu原厂的人来负责，移植好的这些接口文件就放在RVDS这个文件夹的目录下，具体见图11‑8。

|portin009|

图11‑8RVDS文件夹内容

FreeRTOS为我们提供了cortex-m0、m3、m4和m7等内核的单片机的接口文件，只要是使用了这些内核的mcu都可以使用里面的接口文件。通常网络上出现的叫“移植某某某RTOS到某某某MCU”的教程，其实准确来说，不能够叫移植，应该叫使用官方的移植，因为这些跟硬件相关的接口文件，RTOS官方都
已经写好了，我们只是使用而已。我们本章讲的移植也是使用FreeRTOS官方的移植，关于这些底层的移植文件我们已经在第一部分“从0到1教你写FreeRTOS内核”有非常详细的讲解，这里我们直接使用即可。我们这里以ARM_CM3这个文件夹为例，看看里面的文件，里面只有“port.c”与“portmacr
o.h”两个文件，port.c文件里面的内容是由FreeRTOS官方的技术人员为Cortex-M3内核的处理器写的接口文件，里面核心的上下文切换代码是由汇编语言编写而成，对技术员的要求比较高，我们刚开始学习的之后只需拷贝过来用即可，深入的学习可以放在后面的日子；portmacro.h则是port.c
文件对应的头文件，主要是一些数据类型和宏定义，具体见图11‑9。

|portin010|

图11‑9ARM_CM3文件夹内容

编号\ **②**\ MemMang文件夹下存放的是跟内存管理相关的，总共有五个heap文件以及一个readme说明文件，这五个heap文件在移植的时候必须使用一个，因为FreeRTOS在创建内核对象的时候使用的是动态分配内存，而这些动态内存分配的函数则在这几个文件里面实现，不同的分配算法会导致不同
的效率与结果，后面在内存管理中我们会讲解每个文件的区别，由于现在是初学，所以我们选用heap4.c即可，具体见图11‑10。

|portin011|

图11‑10MemMang文件夹内容

至此，FreeRTOS/source文件夹下的主要内容就讲完，剩下的可根据兴趣自行查阅。

Demo文件夹
'''''''

这个目录下内容就是Deme例程，我们可以直接打开里面的工程文件，各种开发平台的完整Demo，开发者可以方便的以此搭建出自己的项目，甚至直接使用。FreeRTOS当然也为ST写了很多Demo，其中就有F1、F4、F7等工程，这样子对我们学习FreeRTOS是非常方便的，当遇到不懂的直接就可以参考官方的
Demo，具体见图11‑11。

|portin012|

图11‑11Demo文件夹内容

License文件夹
''''''''''

这里面只有一个许可文件“license.txt”，用FreeRTOS做产品的话就需要看看这个文件，但是我们是学习FreeRTOS，所以暂时不需要理会这个文件。

FreeRTOS-Plus文件夹
^^^^^^^^^^^^^^^^

FreeRTOS-Plus文件夹里面包含的是第三方的产品，一般我们不需要使用，FreeRTOS-Plus的预配置演示项目组件（组件大多数都要收费），大多数演示项目都是在Windows环境中运行的，使用FreeRTOS windows模拟器，所以暂时不需要关注这个文件夹。

HTML文件
^^^^^^

一些直接可以打开的网页文件，里面包含一些关于FreeRTOS的介绍，是FreeRTOS官方人员所写，所以都是英文的，有兴趣可以打开看看，具体相关内容可以看HTML文件名称。

往裸机工程添加FreeRTOS源码
~~~~~~~~~~~~~~~~~

提取FreeRTOS最简源码
^^^^^^^^^^^^^^

在前一章节中，我们看到了FreeRTOS源码中那么多文件，一开始学我们根本看不过来那么多文件，我们需要提取源码中的最简洁的部分代码，方便同学们学习，更何况我们学习的只是FreeRTOS的实时内核中的知识，因为这才是FreeRTOS的核心，那些demo都是基于此移植而来的，我们不需要学习，下面提取源码
的操作过程。

1. 首先在我们的STM32裸机工程模板根目录下新建一个文件夹，命名为“FreeRTOS”，并且在FreeRTOS文件夹下新建两个空文件夹，分别命名为“src”与“port”，src文件夹用于保存FreeRTOS中的核心源文件，也就是我们常说的‘.c文件’，port文件夹用于保存内存管理以及处理器架构相关
   代码，这些代码FreeRTOS官方已经提供给我们的，直接使用即可，在前面已经说了，FreeRTOS是软件，我们的开发版是硬件，软硬件必须有桥梁来连接，这些与处理器架构相关的代码，可以称之为RTOS硬件接口层，它们位于FreeRTOS/Source/Portable文件夹下。

2. 打开FreeRTOS V9.0.0源码，在“FreeRTOSv9.0.0\FreeRTOS\Source”目录下找到所有的‘.c文件’，将它们拷贝到我们新建的src文件夹中，具体见图11‑12。

..

   |portin013|

图11‑12提取FreeRTOS源码文件（*.c文件）

3. 打开FreeRTOS V9.0.0源码，在“FreeRTOSv9.0.0\FreeRTOS\Source\portable”目录下找到“MemMang”文件夹与“RVDS”文件夹，将它们拷贝到我们新建的port文件夹中，具体见

..

   |portin014|

图11‑13提取MemMang与RVDS源码文件

4. 打开FreeRTOS V9.0.0源码，在“FreeRTOSv9.0.0\\ FreeRTOS\Source”目录下找到“include”文件夹，它是我们需要用到FreeRTOS的一些头文件，将它直接拷贝到我们新建的FreeRTOS文件夹中，完成这一步之后就可以看到我们新建的FreeRTOS文件夹已
   经有3个文件夹，这3个文件夹就包含FreeRTOS的核心文件，至此，FreeRTOS的源码就提取完成，具体见图11‑14。

|portin015|

图11‑14提取FreeRTOS核心文件完成状态

拷贝FreeRTOS到裸机工程根目录
^^^^^^^^^^^^^^^^^^

鉴于FreeRTOS容量很小，我们直接将刚刚提取的整个FreeRTOS文件夹拷贝到我们的STM32裸机工程里面，让整个FreeRTOS跟随我们的工程一起发布，使用这种方法打包的FreeRTOS
工程，即使是将工程拷贝到一台没有安装FreeRTOS支持包（MDK中有FreeRTOS的支持包）的电脑上面都是可以直接使用的，因为工程已经包含了FreeRTOS的源码。具体见图11‑15。

|portin016|

图11‑15拷贝FreeRTOS Package到裸机工程

图11‑15中FreeRTOS文件夹下就是我们提取的FreeRTOS的核心代码，该文件夹下的具体内容作用在前面就已经描述的很清楚了，这里就不再重复赘述。

拷贝FreeRTOSConfig.h文件到user文件夹
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

FreeRTOSConfig.h文件是FreeRTOS的工程配置文件，因为FreeRTOS是可以裁剪的实时操作内核，应用于不同的处理器平台，用户可以通过修改这个FreeRTOS内核的配置头文件来裁剪FreeRTOS的功能，所以我们把它拷贝一份放在user这个文件夹下面。

打开FreeRTOSv9.0.0源码，在“FreeRTOSv9.0.0\FreeRTOS\Demo”文件夹下面找到“CORTEX_STM32F103_Keil”这个文件夹，双击打开，在其根目录下找到这个“FreeRTOSConfig.h”文件，然后拷贝到我们工程的user文件夹下即可，等下我们需要对
这个文件进行修改。user文件夹，见名知义我们就可以知道里面存放的文件都是用户自己编写的。

添加FreeRTOS源码到工程组文件夹
^^^^^^^^^^^^^^^^^^^

在上一步我们只是将FreeRTOS的源码放到了本地工程目录下，还没有添加到开发环境里面的组文件夹里面，FreeRTOS也就没有移植到我们的工程中去。

新建FreeRTOS/src和FreeRTOS/port组
'''''''''''''''''''''''''''''

接下来我们在开发环境里面新建FreeRTOS/src和FreeRTOS/port两个组文件夹，其中FreeRTOS/src用于存放src文件夹的内容，FreeRTOS/port用于存放port\MemMang文件夹与port\RVDS\ARM_CM？文件夹的内容，“？”表示3、4或者7，具体选择哪个
得看你使用的是野火哪个型号的STM32开发板，具体见表11‑1。

表11‑1野火STM32开发板型号对应FreeRTOS的接口文件

=================== ============= ==========================
野火STM32开发板型号 具体芯片型号  FreeRTOS不同内核的接口文件
=================== ============= ==========================
MINI                STM32F103RCT6 port\RVDS\ARM_CM3
指南者              STM32F103VET6
霸道                STM32F103ZET6
霸天虎              STM32F407ZGT6 port\RVDS\ARM_CM4
F429-挑战者         STM32F429IGT6
F767-挑战者         STM32F767IGT6 port\RVDS\ARM_CM7
H743-挑战者         STM32H743IIT6
=================== ============= ==========================

然后我们将工程文件中FreeRTOS的内容添加到工程中去，按照已经新建的分组添加我们的FreeRTOS工程源码。

在FreeRTOS/port分组中添加MemMang文件夹中的文件只需选择其中一个即可，我们选择“heap_4.c”，这是FreeRTOS的一个内存管理源码文件。同时，需要根据自己的开发板型号在FreeRTOS\port\RVDS\ARM_CM?中选择，“？”表示3、4或者7，具体选择哪个得看你使用
的是野火哪个型号的STM32开发板，具体见表11‑1。

然后在user分组中添加我们FreeRTOS的配置文件“FreeRTOSConfig.h”，因为这是头文件（.h），所以需要在添加时选择文件类型为“All files (*.*)”，至此我们的FreeRTOS添加到工程中就已经完成，完成的效果具体见图11‑16。

|portin018|

图11‑16添加FreeRTOS源码到工程分组中

指定FreeRTOS头文件的路径
''''''''''''''''

FreeRTOS的源码已经添加到开发环境的组文件夹下面，编译的时候需要为这些源文件指定头文件的路径，不然编译会报错。FreeRTOS的源码里面只有FreeRTOS\include和FreeRTOS\port\RVDS\ARM_CM？这两个文件夹下面有头文件，只需要将这两个头文件的路径在开发环境里面指
定即可。同时我们还将FreeRTOSConfig.h这个头文件拷贝到了工程根目录下的user文件夹下，所以user的路径也要加到开发环境里面。FreeRTOS头文件的路径添加完成后的效果具体见图11‑17。

|portin018|

图11‑17在开发环境中指定FreeRTOS 的头文件的路径

至此，FreeRTOS的整体工程基本移植完毕，我们需要修改FreeRTOS配置文件，按照我们的需求来进行修改。

修改FreeRTOSConfig.h
~~~~~~~~~~~~~~~~~~

FreeRTOSConfig.h是直接从demo文件夹下面拷贝过来的，该头文件对裁剪整个FreeRTOS所需的功能的宏均做了定义，有些宏定义被使能，有些宏定义被失能，一开始我们只需要配置最简单的功能即可。要想随心所欲的配置FreeRTOS的功能，我们必须对这些宏定义的功能有所掌握，下面我们先简单的介
绍下这些宏定义的含义，然后再对这些宏定义进行修改。

注意：此FreeRTOSConfig.h文件内容与我们从demo移植过来的FreeRTOSConfig.h文件不一样，因为这是我们野火修改过的FreeRTOSConfig.h文件，并不会影响FreeRTOS的功能，我们只是添加了一些中文注释，并且把相关的头文件进行分类，方便查找宏定义已经阅读，仅此而
已。强烈建议使用我们修加工过的FreeRTOSConfig.h文件。

FreeRTOSConfig.h文件内容讲解
^^^^^^^^^^^^^^^^^^^^^^

代码清单11‑1FreeRTOSConfig.h文件内容

1 #ifndef FREERTOS_CONFIG_H

2 #define FREERTOS_CONFIG_H

3

4 //针对不同的编译器调用不同的stdint.h文件

5 #if defined(__ICCARM__) \|\| defined(__CC_ARM) \|\| defined(__GNUC__)\ **(1)**

6 #include <stdint.h>

7 externuint32_t SystemCoreClock;

8 #endif

9

10 //断言

11 #define vAssertCalled(char,int) printf("Error:%s,%d\r\n",char,int)

12 #define configASSERT(x) if((x)==0) vAssertCalled(__FILE__,__LINE__)\ **(2)**

13

14 /\*

15 \* FreeRTOS基础配置配置选项

16 \/

17 /\* 置1：RTOS使用抢占式调度器；置0：RTOS使用协作式调度器（时间片）

18 \*

19 \* 注：在多任务管理机制上，操作系统可以分为抢占式和协作式两种。

20 \* 协作式操作系统是任务主动释放CPU后，切换到下一个任务。

21 \* 任务切换的时机完全取决于正在运行的任务。

22 \*/

23 #define configUSE_PREEMPTION 1 **(3)**

24

25 //1使能时间片调度(默认式使能的)

26 #define configUSE_TIME_SLICING 1 **(4)**

27

28 /\* 某些运行FreeRTOS的硬件有两种方法选择下一个要执行的任务：

29 \* 通用方法和特定于硬件的方法（以下简称“特殊方法”）。

30 \*

31 \* 通用方法：

32 \* 1.configUSE_PORT_OPTIMISED_TASK_SELECTION 为 0 或者硬件不支持这种特殊方法。

33 \* 2.可以用于所有FreeRTOS支持的硬件

34 \* 3.完全用C实现，效率略低于特殊方法。

35 \* 4.不强制要求限制最大可用优先级数目

36 \* 特殊方法：

37 \* 1.必须将configUSE_PORT_OPTIMISED_TASK_SELECTION设置为1。

38 \* 2.依赖一个或多个特定架构的汇编指令（一般是类似计算前导零[CLZ]指令）。

39 \* 3.比通用方法更高效

40 \* 4.一般强制限定最大可用优先级数目为32

41 \*

42 一般是硬件计算前导零指令，如果所使用的，MCU没有这些硬件指令的话此宏应该设置为0！

43 \*/

44 #define configUSE_PORT_OPTIMISED_TASK_SELECTION 1 **(5)**

45

46 /\* 置1：使能低功耗tickless模式；置0：保持系统节拍（tick）中断一直运行 \*/

47 #define configUSE_TICKLESS_IDLE 0 **(6)**

48

49 /\*

50 \* 写入实际的CPU内核时钟频率，也就是CPU指令执行频率，通常称为Fclk

51 \* Fclk为供给CPU内核的时钟信号，我们所说的cpu主频为 XX MHz，

52 \* 就是指的这个时钟信号，相应的，1/Fclk即为cpu时钟周期；

53 \*/

54 #define configCPU_CLOCK_HZ (SystemCoreClock) **(7)**

55

56 //RTOS系统节拍中断的频率。即一秒中断的次数，每次中断RTOS都会进行任务调度

57 #define configTICK_RATE_HZ (( TickType_t )1000) **(8)**

58

59 //可使用的最大优先级

60 #define configMAX_PRIORITIES (32) **(9)**

61

62 //空闲任务使用的栈大小

63 #define configMINIMAL_STACK_SIZE ((unsigned short)128) **(10)**

64

65 //任务名字字符串长度

66 #define configMAX_TASK_NAME_LE (16) **(11)**

67

68 //系统节拍计数器变量数据类型，1表示为16位无符号整形，0表示为32位无符号整形

69 #define configUSE_16_BIT_TICKS 0 **(12)**

70

71 //空闲任务放弃CPU使用权给其他同优先级的用户任务

72 #define configIDLE_SHOULD_YIELD 1 **(13)**

73

74 //启用队列

75 #define configUSE_QUEUE_SETS 1 **(14)**

76

77 //开启任务通知功能，默认开启

78 #define configUSE_TASK_NOTIFICATIONS 1 **(15)**

79

80 //使用互斥信号量

81 #define configUSE_MUTEXES 1 **(16)**

82

83 //使用递归互斥信号量

84 #define configUSE_RECURSIVE_MUTEXES 1 **(17)**

85

86 //为1时使用计数信号量

87 #define configUSE_COUNTING_SEMAPHORES 1 **(18)**

88

89 /\* 设置可以注册的信号量和消息队列个数 \*/

90 #define configQUEUE_REGISTRY_SIZE 10 **(19)**

91

92 #define configUSE_APPLICATION_TASK_TAG 0

93

94

95 /\*

96 FreeRTOS与内存申请有关配置选项

97 \/

98 //支持动态内存申请

99 #define configSUPPORT_DYNAMIC_ALLOCATION 1 **(20)**

100 //支持静态内存

101#define configSUPPORT_STATIC_ALLOCATION 0

102 //系统所有总的堆大小

103 #define configTOTAL_HEAP_SIZE ((size_t)(36*1024)) **(21)**

104 /\*

105 FreeRTOS与钩子函数有关的配置选项

106 \/

107 /\* 置1：使用空闲钩子（Idle Hook类似于回调函数）；置0：忽略空闲钩子

108 \*

109 \* 空闲任务钩子是一个函数，这个函数由用户来实现，

110 \* FreeRTOS规定了函数的名字和参数：void vApplicationIdleHook(void )，

111 \* 这个函数在每个空闲任务周期都会被调用

112 \* 对于已经删除的RTOS任务，空闲任务可以释放分配给它们的栈内存。

113 \* 因此必须保证空闲任务可以被CPU执行

114 \* 使用空闲钩子函数设置CPU进入省电模式是很常见的

115 \* 不可以调用会引起空闲任务阻塞的API函数

116 \*/

117 #define configUSE_IDLE_HOOK 0 **(22)**

118

119 /\* 置1：使用时间片钩子（Tick Hook）；置0：忽略时间片钩子

120 \*

121 \*

122 \* 时间片钩子是一个函数，这个函数由用户来实现，

123 \* FreeRTOS规定了函数的名字和参数：void vApplicationTickHook(void )

124 \* 时间片中断可以周期性的调用

125 \* 函数必须非常短小，不能大量使用栈，

126 \* 不能调用以”FromISR" 或 "FROM_ISR”结尾的API函数

127 \*/

128 #define configUSE_TICK_HOOK 0 **(23)**

129

130 //使用内存申请失败钩子函数

131 #define configUSE_MALLOC_FAILED_HOOK 0 **(24)**

132

133 /\*

134 \* 大于0时启用栈溢出检测功能，如果使用此功能

135 \* 用户必须提供一个栈溢出钩子函数，如果使用的话

136 \* 此值可以为1或者2，因为有两种栈溢出检测方法 \*/

137 #define configCHECK_FOR_STACK_OVERFLOW 0 **(25)**

138

139

140 /\*

141 FreeRTOS与运行时间和任务状态收集有关的配置选项

142 \/

143 //启用运行时间统计功能

144 #define configGENERATE_RUN_TIME_STATS 0 **(26)**

145 //启用可视化跟踪调试

146 #define configUSE_TRACE_FACILITY 0 **(27)**

147 /\* 与宏configUSE_TRACE_FACILITY同时为1时会编译下面3个函数

148 \* prvWriteNameToBuffer()

149 \* vTaskList(),

150 \* vTaskGetRunTimeStats()

151 \*/

152 #define configUSE_STATS_FORMATTING_FUNCTIONS 1

153

154

155 /\*

156 FreeRTOS与协程有关的配置选项

157 \/

158 //启用协程，启用协程以后必须添加文件croutine.c

159 #define configUSE_CO_ROUTINES 0 **(28)**

160 //协程的有效优先级数目

161 #define configMAX_CO_ROUTINE_PRIORITIES ( 2 ) **(29)**

162

163

164 /\*

165 FreeRTOS与软件定时器有关的配置选项

166 \/

167 //启用软件定时器

168 #define configUSE_TIMERS 1 **(30)**

169 //软件定时器优先级

170 #define configTIMER_TASK_PRIORITY (configMAX_PRIORITIES-1) **(31)**

171 //软件定时器队列长度

172 #define configTIMER_QUEUE_LENGTH 10 **(32)**

173 //软件定时器任务栈大小

174 #define configTIMER_TASK_STACK_DEPTH (configMINIMAL_STACK_SIZE*2)\ **(33)**

175

176 /\*

177 FreeRTOS可选函数配置选项

178 \/

179 #define INCLUDE_xTaskGetSchedulerState 1 **(34)**

180 #define INCLUDE_vTaskPrioritySet 1 **(35)**

181 #define INCLUDE_uxTaskPriorityGet 1 **(36)**

182 #define INCLUDE_vTaskDelete 1 **(37)**

183 #define INCLUDE_vTaskCleanUpResources 1

184 #define INCLUDE_vTaskSuspend 1

185 #define INCLUDE_vTaskDelayUntil 1

186 #define INCLUDE_vTaskDelay 1

187 #define INCLUDE_eTaskGetState 1

188 #define INCLUDE_xTimerPendFunctionCall 1

189

190 /\*

191 FreeRTOS与中断有关的配置选项

192 \/

193 #ifdef \__NVIC_PRIO_BITS

194 #define configPRIO_BITS \__NVIC_PRIO_BITS **(38)**

195 #else

196 #define configPRIO_BITS 4 **(39)**

197 #endif

198 //中断最低优先级

199 #define configLIBRARY_LOWEST_INTERRUPT_PRIORITY 15 **(40)**

200

201 //系统可管理的最高中断优先级

202 #define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5 **(41)**

203 #define configKERNEL_INTERRUPT_PRIORITY **(42)**

204 ( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )

205

206 #define configMAX_SYSCALL_INTERRUPT_PRIORITY **(43)**

207 ( configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )

208 /\*

209 FreeRTOS与中断服务函数有关的配置选项

210 \/

211 #define xPortPendSVHandler PendSV_Handler

212 #define vPortSVCHandler SVC_Handler

213

214 /\* 以下为使用Percepio Tracealyzer需要的东西，不需要时将

215 configUSE_TRACE_FACILITY 定义为 0 \*/

216 #if ( configUSE_TRACE_FACILITY == 1 ) **(44)**

217 #include"trcRecorder.h"

218 #define INCLUDE_xTaskGetCurrentTaskHandle 0

219 // 启用一个可选函数（该函数被Trace源码使用，默认该值为0 表示不用）

220 #endif

221

222

223 #endif/\* FREERTOS_CONFIG_H \*/

224

代码清单11‑1\ **(1)**\ ：针对不同的编译器调用不同的stdint.h文件，在MDK中，我们默认的是__CC_ARM。

代码清单11‑1\ **(2)**\ ：断言简介：在使用C语言编写工程代码时，我们总会对某种假设条件进行检查，断言就是用于在代码中捕捉这些假设，可以将断言看作是异常处理的一种高级形式。断言表示为一些布尔表达式，程序员相信在程序中的某个特定表达式值为真。可以在任何时候启用和禁用断言验证，因此可以在测试
时启用断言，而在发布时禁用断言。同样，程序投入运行后，最终用户在遇到问题时可以重新启用断言。它可以快速发现并定位软件问题，同时对系统错误进行自动报警。断言可以对在系统中隐藏很深，用其他手段极难发现的问题可以用断言来进行定位，从而缩短软件问题定位时间，提高系统的可测性。实际应用时，可根据具体情况灵活地
设计断言。这里只是使用宏定义实现了断言的功能，断言作用很大，特别是在调试的时候，而FreeRTOS中使用了很多断言接口configASSERT，所以我们需要实现断言，把错误信息打印出来从而在调试中快速定位，打印信息的内容是xxx文件xxx行(__FILE__,__LINE__)。

代码清单11‑1\ **(3)**\ ：置1：FreeRTOS使用抢占式调度器；置0：FreeRTOS使用协作式调度器（时间片）。抢占式调度：在这种调度方式中，系统总是选择优先级最高的任务进行调度，并且一旦高优先级的任务准备就绪之后，它就会马上被调度而不等待低优先级的任务主动放弃CPU，高优先级的任
务抢占了低优先级任务的CPU使用权，这就是抢占，在实习操作系统中，这样子的方式往往是最适用的。而协作式调度则是由任务主动放弃CPU，然后才进行任务调度。

注意：在多任务管理机制上，操作系统可以分为抢占式和协作式两种。协作式操作系统是任务主动释放CPU后，切换到下一个任务。任务切换的时机完全取决于正在运行的任务。

代码清单11‑1\ **(4)**\ ：使能时间片调度(默认式使能的)。当优先级相同的时候，就会采用时间片调度，这意味着RTOS调度器总是运行处于最高优先级的就绪任务，在每个FreeRTOS系统节拍中断时在相同优先级的多个任务间进行任务切换。如果宏configUSE_TIME_SLICING设置为0
，FreeRTOS调度器仍然总是运行处于最高优先级的就绪任务，但是当RTOS 系统节拍中断发生时，相同优先级的多个任务之间不再进行任务切换，而是在执行完高优先级的任务之后才进行任务切换。一般来说，FreeRTOS默认支持32个优先级，很少情况会把32个优先级全用完，所以，官方建议采用抢占式调度。

代码清单11‑1\ **(5)**\ ：FreeRTOS支持两种方法选择下一个要执行的任务：一个是软件方法扫描就绪链表，这种方法我们通常称为通用方法，configUSE_PORT_OPTIMISED_TASK_SELECTION 为 0 或者硬件不支持特殊方法，才使用通用方法获取下一个即将运行的任务
，通用方法可以用于所有FreeRTOS支持的硬件平台，因为这种方法是完全用C语言实现，所以效率略低于特殊方法，但不强制要求限制最大可用优先级数目；另一个是硬件方式查找下一个要运行的任务，必须将configUSE_PORT_OPTIMISED_TASK_SELECTION设置为1，因为是必须依赖一个或
多个特定架构的汇编指令（一般是类似计算前导零[CLZ]指令，在M3、M4、M7内核中都有，这个指令是用来计算一个变量从最高位开始的连续零的个数），所以效率略高于通用方法，但受限于硬件平台，一般强制限定最大可用优先级数目为32，这也是FreeRTOS官方为什么推荐使用32位优先级的原因。

代码清单11‑1\ **(6)**\ ：低功耗tickless模式。置1：使能低功耗tickless模式；置0：保持系统节拍（tick）中断一直运行，如果不是用于低功耗场景，我们一般置0即可。

代码清单11‑1\ **(7)**\ ：配置CPU内核时钟频率，也就是CPU指令执行频率，通常称为Fclk ， Fclk为供给CPU内核的时钟信号，我们所说的cpu主频为 XX
MHz，就是指的这个时钟信号，相应的，1/Fclk即为CPU时钟周期，在野火STM32霸道开发板上系统时钟为SystemCoreClock = SYSCLK_FREQ_72MHz，也就是72MHz。

代码清单11‑1\ **(8)**\ ：FreeRTOS系统节拍中断的频率。表示操作系统每1秒钟产生多少个tick，tick即是操作系统节拍的时钟周期，时钟节拍就是系统以固定的频率产生中断（时基中断），并在中断中处理与时间相关的事件，推动所有任务向前运行。时钟节拍需要依赖于硬件定时器，在STM32
裸机程序中经常使用的SysTick 时钟是MCU的内核定时器，通常都使用该定时器产生操作系统的时钟节拍。在FreeRTOS中，系统延时和阻塞时间都是以tick为单位，配置configTICK_RATE_HZ的值可以改变中断的频率，从而间接改变了FreeRTOS的时钟周期（T=1/f）。我们设置为10
00，那么FreeRTOS的时钟周期为1ms，过高的系统节拍中断频率也意味着FreeRTOS内核占用更多的CPU时间，因此会降低效率，一般配置为100~1000即可。

代码清单11‑1\ **(9)**\ ：可使用的最大优先级，默认为32即可，官方推荐的也是32。每一个任务都必须被分配一个优先级，优先级值从0~ （configMAX_PRIORITIES - 1）之间。低优先级数值表示低优先级任务。空闲任务的优先级为0（tskIDLE_PRIORITY），因此它是
最低优先级任务。FreeRTOS调度器将确保处于就绪态的高优先级任务比同样处于就绪状态的低优先级任务优先获取处理器时间。换句话说，FreeRTOS运行的永远是处于就绪态的高优先级任务。处于就绪状态的相同优先级任务使用时间片调度机制共享处理器时间。

代码清单11‑1\ **(10)**\ ：空闲任务默认使用的栈大小，默认为128字即可（在M3、M4、M7中为128*4字节），栈大小不是以字节为单位而是以字为单位的，比如在32位架构下，栈大小为100表示栈内存占用400字节的空间。

代码清单11‑1\ **(11)**\ ：任务名字字符串长度，这个宏用来定义该字符串的最大长度。这里定义的长度包括字符串结束符’\0’。

代码清单11‑1\ **(12)**\ ：系统节拍计数器变量数据类型，1表示为16位无符号整形，0表示为32位无符号整形，STM32是32位机器，所以默认使用为0即可，这个值位数的大小决定了能计算多少个tick，比如假设系统以1ms产生一个tick中断的频率计时，那么32位无符号整形的值则可以计算4
294967295个tick，也就是系统从0运行到4294967.295秒的时候才溢出，转换为小时的话，则能运行1193个小时左右才溢出，当然，溢出就会重置时间，这点完全不用担心；而假如使用16位无符号整形的值，只能计算65535个tick，在65.535秒之后就会溢出，然后重置。

代码清单11‑1\ **(13)**\
：控制任务在空闲优先级中的行为，空闲任务放弃CPU使用权给其他同优先级的用户任务。仅在满足下列条件后，才会起作用，1：启用抢占式调度；2：用户任务优先级与空闲任务优先级相等。一般不建议使用这个功能，能避免尽量避免，1：设置用户任务优先级比空闲任务优先级高，2：这个宏定义配置为0。

代码清单11‑1\ **(14)**\ ：启用消息队列，消息队列是FreeRTOS的IPC通信的一种，用于传递消息。

代码清单11‑1\ **(15)**\ ：开启任务通知功能，默认开启。每个FreeRTOS任务具有一个32位的通知值，FreeRTOS任务通知是直接向任务发送一个事件，并且接收任务的通知值是可以选择的，任务通过接收到的任务通知值来解除任务的阻塞状态（假如因等待该任务通知而进入阻塞状态）。相对于队列、
二进制信号量、计数信号量或事件组等IPC通信，使用任务通知显然更灵活。官方说明：相比于使用信号量解除任务阻塞，使用任务通知可以快45%（使用GCC编译器，-o2优化级别），并且使用更少的RAM。

FreeRTOS官方说明：Unblocking an RTOS task with a direct notification is 45% faster and uses less RAM than unblocking a task with a binary semaphore.

代码清单11‑1\ **(16)**\ ：使用互斥信号量。

代码清单11‑1\ **(17)**\ ：使用递归互斥信号量。

代码清单11‑1\ **(18)**\ ：使用计数信号量。

代码清单11‑1\ **(19)**\ ：设置可以注册的信号量和消息队列个数，用户可以根据自己需要修改即可，RAM小的芯片尽量裁剪得小一些。

代码清单11‑1\ **(20)**\ ：支持动态分配申请，一般在系统中采用的内存分配都是动态内存分配。FreeRTOS同时也支持静态分配内存，但是常用的就是动态分配了。

代码清单11‑1\ **(21)**\ ： FreeRTOS内核总计可用的有效的RAM大小，不能超过芯片的RAM大小，一般来说用户可用的内存大小会小于configTOTAL_HEAP_SIZE定义的大小，因为系统本身就需要内存。每当创建任务、队列、互斥量、软件定时器或信号量时，FreeRTOS内核会
为这些内核对象分配RAM，这里的RAM都属于configTOTAL_HEAP_SIZE指定的内存区。

代码清单11‑1\ **(22)**\ ：配置空闲钩子函数，钩子函数是类似一种回调函数，在任务执行到某个点的时候，跳转到对应的钩子函数执行，这个宏定义表示是否启用空闲任务钩子函数，这个函数由用户来实现，但是FreeRTOS规定了函数的名字和参数：void
vApplicationIdleHook(void)，我们自定义的钩子函数不允许出现阻塞的情况。

代码清单11‑1\ **(23)**\ ：配置时间片钩子函数，与空闲任务钩子函数一样。这个宏定义表示是否启用时间片钩子函数，这个函数由用户来实现，但是FreeRTOS规定了函数的名字和参数：void vApplicationTickHook(void)，我们自定义的钩子函数不允许出现阻塞的情况。同时
需要知道的是xTaskIncrementTick函数在xPortSysTickHandler中断函数中被调用的。因此，vApplicationTickHook()函数执行的时间必须很短才行，同时不能调用任何不是以”FromISR" 或 "FROM_ISR”结尾的API函数。

代码清单11‑1\ **(24)**\ ：使用内存申请失败钩子函数。

代码清单11‑1\ **(25)**\ ：这个宏定义大于0时启用栈溢出检测功能，如果使用此功能，用户必须提供一个栈溢出钩子函数，如果使用的话，此值可以为1或者2，因为有两种栈溢出检测方法。使用该功能，可以分析是否有内存越界的情况。

代码清单11‑1\ **(26)**\ ：不启用运行时间统计功能。

代码清单11‑1\ **(27)**\ ：启用可视化跟踪调试。

代码清单11‑1\ **(28)**\ ：启用协程，启用协程以后必须添加文件croutine.c，默认不使用，因为FreeRTOS不对协程做支持了。

代码清单11‑1\ **(29)**\ ：协程的有效优先级数目，当configUSE_CO_ROUTINES这个宏定义有效的时候才有效，默认即可。

代码清单11‑1\ **(30)**\ ：启用软件定时器。

代码清单11‑1\ **(31)**\ ：配置软件定时器任务优先级为最高优先级(configMAX_PRIORITIES-1) 。

代码清单11‑1\ **(32)**\ ：软件定时器队列长度，也就是允许配置多少个软件定时器的数量，其实FreeRTOS中理论上能配置无数个软件定时器，因为软件定时器是不基于硬件的。

代码清单11‑1\ **(33)**\ ：配置软件定时器任务栈大小，默认为(configMINIMAL_STACK_SIZE*2)。

代码清单11‑1\ **(34)**\ ：必须将INCLUDE_XTaskGetSchedulerState这个宏定义必须设置为1才能使用xTaskGetSchedulerState()这个API函数接口。

代码清单11‑1\ **(35)**\ ：INCLUDE_VTaskPrioritySet这个宏定义必须设置为1才能使vTaskPrioritySet()这个API函数接口。

代码清单11‑1\ **(36)**\ ：INCLUDE_uxTaskPriorityGet这个宏定义必须设置为1才能使uxTaskPriorityGet()这个API函数接口。

代码清单11‑1\ **(37)**\ ：INCLUDE_vTaskDelete这个宏定义必须设置为1才能使vTaskDelete()这个API函数接口。其他都是可选的宏定义，根据需要自定义即可。

代码清单11‑1\ **(38)**\ ：定义__NVIC_PRIO_BITS表示配置FreeRTOS使用多少位作为中断优先级，在STM32中使用4位作为中断的优先级。

代码清单11‑1\ **(39)**\ ：如果没有定义，那么默认就是4位。

代码清单11‑1\ **(40)**\
：配置中断最低优先级是15（一般配置为15）。configLIBRARY_LOWEST_INTERRUPT_PRIORITY是用于配置SysTick与PendSV的。注意了：这里是中断优先级，中断优先级的数值越小，优先级越高。而FreeRTOS的任务优先级是，任务优先级数值越小，任务优先级越低。

代码清单11‑1\ **(41)**\ ：配置系统可管理的最高中断优先级为5，configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY是用于配置basepri寄存器的，当basepri设置为某个值的时候，会让系统不响应比该优先级低的中断，而优先级比之更高的中断则不受影
响。就是说当这个宏定义配置为5的时候，中断优先级数值在0、1、2、3、4的这些中断是不受FreeRTOS管理的，不可被屏蔽，也不能调用FreeRTOS中的API函数接口，而中断优先级在5到15的这些中断是受到系统管理，可以被屏蔽的。

代码清单11‑1\ **(42)**\ ：对需要配置的SysTick与PendSV进行偏移（因为是高4位才有效），在port.c中会用到configKERNEL_INTERRUPT_PRIORITY这个宏定义来配置SCB_SHPR3（系统处理优先级寄存器，地址为：0xE000
ED20），具体见图11‑18。

|portin019|

图11‑18配置SysTick与PendSV（高4位才可读）

代码清单11‑1\ **(43)**\ ：configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY是用于配置basepri寄存器的，让FreeRTOS屏蔽优先级数值大于这个宏定义的中断（数值越大，优先级越低），而basepri的有效位为高4位，所以需要进行偏移，因为S
TM32只使用了优先级寄存器中的4位，所以要以最高有效位对齐，具体见图11‑19。

还需要注意的是：中断优先级0（具有最高的逻辑优先级）不能被basepri寄存器屏蔽，因此，configMAX_SYSCALL_INTERRUPT_PRIORITY绝不可以设置成0。

|portin020|

图11‑19配置basepri寄存器

为什么要屏蔽中断?

先了解一下什么是临界段！临界段用一句话概括就是一段在执行的时候不能被中断的代码段。在FreeRTOS里面，这个临界段最常出现的就是对全局变量的操作，全局变量就好像是一个枪把子，谁都可以对他开枪，但是我开枪的时候，你就不能开枪，否则就不知道是谁命中了靶子。

那么什么情况下临界段会被打断？一个是系统调度，还有一个就是外部中断。在FreeRTOS中，系统调度，最终也是产生PendSV中断，在PendSV
Handler里面实现任务的切换，所以还是可以归结为中断。既然这样，FreeRTOS对临界段的保护就很有必要了，在必要的时候将中断屏蔽掉，但是又必须保证某些特别紧急的中断的处理，比如像无人机的碰撞检测。

PRIMASK和FAULTMAST是Cortex-M内核里面三个中断屏蔽寄存器中的两个，还有一个是BASEPRI，有关这三个寄存器的详细用法见表11‑2。

表11‑2Cortex-M内核中断屏蔽寄存器组描述

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


代码清单11‑1\ **(44)**\ ：configUSE_TRACE_FACILITY这个宏定义是用于FreeRTOS可视化调试软件Tracealyzer需要的东西，我们现在暂时不需要，将 configUSE_TRACE_FACILITY 定义为 0即可。

FreeRTOSConfig.h文件修改
^^^^^^^^^^^^^^^^^^^^

FreeRTOSConfig.h头文件的内容修改的不多，具体是：修改与对应开发板的头文件，如果是使用野火STM32F1的开发板，则包含F1的头文件#include
"stm32f10x.h"，同理是使用了其他系列的开发板，则包含与开发板对应的头文件即可，当然还需要包含我们的串口的头文件“bsp_usart.h”，因为在我们FreeRTOSConfig.h中实现了断言操作，需要打印一些信息。其他根据需求修改即可，具体见代码清单11‑2的加粗部分。

提示：虽然FreeRTOS中默认是打开很多宏定义的，但是用户还是要根据需要选择打开与关闭，因为这样子的系统会更适合用户需要，更严谨与更加节省系统资源。

代码清单11‑2rtconfig.h文件修改

1 #ifndef FREERTOS_CONFIG_H

2 #define FREERTOS_CONFIG_H

3

4

**5 #include"stm32f10x.h"**

**6 #include"bsp_usart.h"**

7

8

9 //针对不同的编译器调用不同的stdint.h文件

10 #if defined(__ICCARM__) \|\| defined(__CC_ARM) \|\| defined(__GNUC__)

11 #include <stdint.h>

12 externuint32_t SystemCoreClock;

13 #endif

14

15 //断言

16 #define vAssertCalled(char,int) printf("Error:%s,%d\r\n",char,int)

17 #define configASSERT(x) if((x)==0) vAssertCalled(__FILE__,__LINE__)

18

19 /\*

20 \* FreeRTOS基础配置配置选项

21 \/

22 /\* 置1：RTOS使用抢占式调度器；置0：RTOS使用协作式调度器（时间片）

23 \*

24 \* 注：在多任务管理机制上，操作系统可以分为抢占式和协作式两种。

25 \* 协作式操作系统是任务主动释放CPU后，切换到下一个任务。

26 \* 任务切换的时机完全取决于正在运行的任务。

27 \*/

28 #define configUSE_PREEMPTION 1

29

30 //1使能时间片调度(默认式使能的)

31 #define configUSE_TIME_SLICING 1

32

33 /\* 某些运行FreeRTOS的硬件有两种方法选择下一个要执行的任务：

34 \* 通用方法和特定于硬件的方法（以下简称“特殊方法”）。

35 \*

36 \* 通用方法：

37 \* 1.configUSE_PORT_OPTIMISED_TASK_SELECTION 为 0 或者硬件不支持这种特殊方法。

38 \* 2.可以用于所有FreeRTOS支持的硬件

39 \* 3.完全用C实现，效率略低于特殊方法。

40 \* 4.不强制要求限制最大可用优先级数目

41 \* 特殊方法：

42 \* 1.必须将configUSE_PORT_OPTIMISED_TASK_SELECTION设置为1。

43 \* 2.依赖一个或多个特定架构的汇编指令（一般是类似计算前导零[CLZ]指令）。

44 \* 3.比通用方法更高效

45 \* 4.一般强制限定最大可用优先级数目为32

46 \*

47 一般是硬件计算前导零指令，如果所使用的，MCU没有这些硬件指令的话此宏应该设置为0！

48 \*/

49 #define configUSE_PORT_OPTIMISED_TASK_SELECTION 1

50

51 /\* 置1：使能低功耗tickless模式；置0：保持系统节拍（tick）中断一直运行 \*/

52 #define configUSE_TICKLESS_IDLE 1

53

54 /\*

55 \* 写入实际的CPU内核时钟频率，也就是CPU指令执行频率，通常称为Fclk

56 \* Fclk为供给CPU内核的时钟信号，我们所说的cpu主频为 XX MHz，

57 \* 就是指的这个时钟信号，相应的，1/Fclk即为cpu时钟周期；

58 \*/

59 #define configCPU_CLOCK_HZ (SystemCoreClock)

60

61 //RTOS系统节拍中断的频率。即一秒中断的次数，每次中断RTOS都会进行任务调度

62 #define configTICK_RATE_HZ (( TickType_t )1000)

63

64 //可使用的最大优先级

65 #define configMAX_PRIORITIES (32)

66

67 //空闲任务使用的栈大小

68 #define configMINIMAL_STACK_SIZE ((unsigned short)128)

69

70 //任务名字字符串长度

71 #define configMAX_TASK_NAME_LEN (16)

72

73 //系统节拍计数器变量数据类型，1表示为16位无符号整形，0表示为32位无符号整形

74 #define configUSE_16_BIT_TICKS 0

75

76 //空闲任务放弃CPU使用权给其他同优先级的用户任务

77 #define configIDLE_SHOULD_YIELD 1

78

79 //启用队列

80 #define configUSE_QUEUE_SETS 1

81

82 //开启任务通知功能，默认开启

83 #define configUSE_TASK_NOTIFICATIONS 1

84

85 //使用互斥信号量

86 #define configUSE_MUTEXES 1

87

88 //使用递归互斥信号量

89 #define configUSE_RECURSIVE_MUTEXES 1

90

91 //为1时使用计数信号量

92 #define configUSE_COUNTING_SEMAPHORES 1

93

94 /\* 设置可以注册的信号量和消息队列个数 \*/

95 #define configQUEUE_REGISTRY_SIZE 10

96

97 #define configUSE_APPLICATION_TASK_TAG 0

98

99

100 /\*

101 FreeRTOS与内存申请有关配置选项

102 \/

103 //支持动态内存申请

104 #define configSUPPORT_DYNAMIC_ALLOCATION 1

105 //系统所有总的堆大小

106 #define configTOTAL_HEAP_SIZE ((size_t)(36*1024))

107

108

109 /\*

110 FreeRTOS与钩子函数有关的配置选项

111 \/

112 /\* 置1：使用空闲钩子（Idle Hook类似于回调函数）；置0：忽略空闲钩子

113 \*

114 \* 空闲任务钩子是一个函数，这个函数由用户来实现，

115 \* FreeRTOS规定了函数的名字和参数：void vApplicationIdleHook(void )，

116 \* 这个函数在每个空闲任务周期都会被调用

117 \* 对于已经删除的RTOS任务，空闲任务可以释放分配给它们的栈内存。

118 \* 因此必须保证空闲任务可以被CPU执行

119 \* 使用空闲钩子函数设置CPU进入省电模式是很常见的

120 \* 不可以调用会引起空闲任务阻塞的API函数

121 \*/

122 #define configUSE_IDLE_HOOK 0

123

124 /\* 置1：使用时间片钩子（Tick Hook）；置0：忽略时间片钩子

125 \*

126 \*

127 \* 时间片钩子是一个函数，这个函数由用户来实现，

128 \* FreeRTOS规定了函数的名字和参数：void vApplicationTickHook(void )

129 \* 时间片中断可以周期性的调用

130 \* 函数必须非常短小，不能大量使用栈，

131 \* 不能调用以”FromISR" 或 "FROM_ISR”结尾的API函数

132 \*/

133 /*xTaskIncrementTick函数是在xPortSysTickHandler中断函数中被调用的。因此，

134 \* vApplicationTickHook()函数执的时间必须很短才行

135 \*/

136

137

138 #define configUSE_TICK_HOOK 0

139

140 //使用内存申请失败钩子函数

141 #define configUSE_MALLOC_FAILED_HOOK 0

142

143 /\*

144 \* 大于0时启用栈溢出检测功能，如果使用此功能

145 \* 用户必须提供一个栈溢出钩子函数，如果使用的话

146 \* 此值可以为1或者2，因为有两种栈溢出检测方法 \*/

147 #define configCHECK_FOR_STACK_OVERFLOW 0

148

149

150 /\*

151 FreeRTOS与运行时间和任务状态收集有关的配置选项

152 \/

153 //启用运行时间统计功能

154 #define configGENERATE_RUN_TIME_STATS 0

155 //启用可视化跟踪调试

156 #define configUSE_TRACE_FACILITY 0

157 /\* 与宏configUSE_TRACE_FACILITY同时为1时会编译下面3个函数

158 \* prvWriteNameToBuffer()

159 \* vTaskList(),

160 \* vTaskGetRunTimeStats()

161 \*/

162 #define configUSE_STATS_FORMATTING_FUNCTIONS 1

163

164

165 /\*

166 FreeRTOS与协程有关的配置选项

167 \/

168 //启用协程，启用协程以后必须添加文件croutine.c

169 #define configUSE_CO_ROUTINES 0

170 //协程的有效优先级数目

171 #define configMAX_CO_ROUTINE_PRIORITIES ( 2 )

172

173

174 /\*

175 FreeRTOS与软件定时器有关的配置选项

176 \/

177 //启用软件定时器

178 #define configUSE_TIMERS 1

179 //软件定时器优先级

180 #define configTIMER_TASK_PRIORITY (configMAX_PRIORITIES-1)

181 //软件定时器队列长度

182 #define configTIMER_QUEUE_LENGTH 10

183 //软件定时器任务栈大小

184 #define configTIMER_TASK_STACK_DEPTH (configMINIMAL_STACK_SIZE*2)

185

186 /\*

187 FreeRTOS可选函数配置选项

188 \/

189 #define INCLUDE_xTaskGetSchedulerState 1

190 #define INCLUDE_vTaskPrioritySet 1

191 #define INCLUDE_uxTaskPriorityGet 1

192 #define INCLUDE_vTaskDelete 1

193 #define INCLUDE_vTaskCleanUpResources 1

194 #define INCLUDE_vTaskSuspend 1

195 #define INCLUDE_vTaskDelayUntil 1

196 #define INCLUDE_vTaskDelay 1

197 #define INCLUDE_eTaskGetState 1

198 #define INCLUDE_xTimerPendFunctionCall 1

199

200 /\*

201 FreeRTOS与中断有关的配置选项

202 \/

203 #ifdef \__NVIC_PRIO_BITS

204 #define configPRIO_BITS \__NVIC_PRIO_BITS

205 #else

206 #define configPRIO_BITS 4

207 #endif

208 //中断最低优先级

209 #define configLIBRARY_LOWEST_INTERRUPT_PRIORITY 15

210

211 //系统可管理的最高中断优先级

212 #define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5

213

214 #define configKERNEL_INTERRUPT_PRIORITY /\* 240 \*/

215 ( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )

216 #define configMAX_SYSCALL_INTERRUPT_PRIORITY

217 ( configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )

218

219 /\*

220 FreeRTOS与中断服务函数有关的配置选项

221 \/

**222 #define xPortPendSVHandler PendSV_Handler**

**223 #define vPortSVCHandler SVC_Handler**

224

225

226 /\* 以下为使用Percepio Tracealyzer需要的东西，不需要时将 configUSE_TRACE_FACILITY 定义为 0 \*/

227 #if ( configUSE_TRACE_FACILITY == 1 )

228 #include"trcRecorder.h"

229// 启用一个可选函数（该函数被Trace源码使用，默认该值为0 表示不用）

230 #define INCLUDE_xTaskGetCurrentTaskHandle 1

231 #endif

232

233

234 #endif/\* FREERTOS_CONFIG_H \*/

235

修改stm32f10x_it.c
~~~~~~~~~~~~~~~~

SysTick中断服务函数是一个非常重要的函数，FreeRTOS所有跟时间相关的事情都在里面处理，SysTick就是FreeRTOS的一个心跳时钟，驱动着FreeRTOS的运行，就像人的心跳一样，假如没有心跳，我们就相当于“死了”，同样的，FreeRTOS没有了心跳，那么它就会卡死在某个地方，不能进
行任务调度，不能运行任何的东西，因此我们需要实现一个FreeRTOS的心跳时钟，FreeRTOS帮我们实现了SysTick的启动的配置：在port.c文件中已经实现vPortSetupTimerInterrupt()函数，并且FreeRTOS通用的SysTick中断服务函数也实现了：在port.c文
件中已经实现xPortSysTickHandler()函数，所以移植的时候只需要我们在stm32f10x_it.c文件中实现我们对应（STM32）平台上的SysTick_Handler()函数即可。FreeRTOS为开发者考虑得特别多，PendSV_Handler()与SVC_Handler()这两
个很重要的函数都帮我们实现了，在在port.c文件中已经实现xPortPendSVHandler()与vPortSVCHandler()函数，防止我们自己实现不了，那么在stm32f10x_it.c中就需要我们注释掉PendSV_Handler()与SVC_Handler()这两个函数了，具体实现见
代码清单11‑3加粗部分。

代码清单11‑3stm32f10x_it.c文件内容

1 /\* Includes -------------------------------------------------------*/

2 #include"stm32f10x_it.h"

3 //FreeRTOS使用

4 #include"FreeRTOS.h"

5 #include"task.h"

6

7 /*\* @addtogroup STM32F10x_StdPeriph_Template

8 \* @{

9 \*/

10

11 /\* Private typedef ----------------------------------------------*/

12 /\* Private define ------------------------------------------------*/

13 /\* Private macro -------------------------------------------------*/

14 /\* Private variables ---------------------------------------------*/

15 /\* Private function prototypes ---------------------------------*/

16 /\* Private functions ----------------------------------------------*/

17

18 //

19 /\* Cortex-M3 Processor Exceptions Handlers \*/

20 //

21

22 /*\*

23 \* @brief This function handles NMI exception.

24 \* @param None

25 \* @retval None

26 \*/

27 void NMI_Handler(void)

28 {

29 }

30

31 /*\*

32 \* @brief This function handles Hard Fault exception.

33 \* @param None

34 \* @retval None

35 \*/

36 void HardFault_Handler(void)

37 {

38 /\* Go to infinite loop when Hard Fault exception occurs \*/

39 while (1) {

40 }

41 }

42

43 /*\*

44 \* @brief This function handles Memory Manage exception.

45 \* @param None

46 \* @retval None

47 \*/

48 void MemManage_Handler(void)

49 {

50 /\* Go to infinite loop when Memory Manage exception occurs \*/

51 while (1) {

52 }

53 }

54

55 /*\*

56 \* @brief This function handles Bus Fault exception.

57 \* @param None

58 \* @retval None

59 \*/

60 void BusFault_Handler(void)

61 {

62 /\* Go to infinite loop when Bus Fault exception occurs \*/

63 while (1) {

64 }

65 }

66

67 /*\*

68 \* @brief This function handles Usage Fault exception.

69 \* @param None

70 \* @retval None

71 \*/

72 void UsageFault_Handler(void)

73 {

74 /\* Go to infinite loop when Usage Fault exception occurs \*/

75 while (1) {

76 }

77 }

78

79 /*\*

80 \* @brief This function handles SVCall exception.

81 \* @param None

82 \* @retval None

83 \*/

**84 //void SVC_Handler(void)**

**85 //{**

**86 //}**

87

88 /*\*

89 \* @brief This function handles Debug Monitor exception.

90 \* @param None

91 \* @retval None

92 \*/

93 void DebugMon_Handler(void)

94 {

95 }

96

97 /*\*

98 \* @brief This function handles PendSVC exception.

99 \* @param None

100 \* @retval None

101 \*/

**102 //void PendSV_Handler(void)**

**103 //{**

**104 //}**

105

106 ///*\*

107 // \* @brief This function handles SysTick Handler.

108 // \* @param None

109 // \* @retval None

110 // \*/

111 externvoid xPortSysTickHandler(void);

**112 //systick中断服务函数**

**113 void SysTick_Handler(void)**

**114 {**

**115 #if (INCLUDE_xTaskGetSchedulerState == 1 )**

**116 if (xTaskGetSchedulerState() != taskSCHEDULER_NOT_STARTED) {**

**117 #endif/\* INCLUDE_xTaskGetSchedulerState \*/**

**118 xPortSysTickHandler();**

**119 #if (INCLUDE_xTaskGetSchedulerState == 1 )**

**120 }**

**121 #endif/\* INCLUDE_xTaskGetSchedulerState \*/**

**122 }**

123

124 //

125 /\* STM32F10x Peripherals Interrupt Handlers \*/

126 /\* Add here the Interrupt Handler for the used peripheral(s) (PPP), for the \*/

127 /\* available peripheral interrupt handler's name please refer to the startup \*/

128 /\* file (startup_stm32f10x_xx.s).
\*/

129 //

130

131 /*\*

132 \* @brief This function handles PPP interrupt request.

133 \* @param None

134 \* @retval None

135 \*/

136 /*void PPP_IRQHandler(void)

137 {

138 }*/

139

140 /*\*

141 \* @}

142 \*/

143

144

145 /\* (C) COPYRIGHT 2011 STMicroelectronics \END OF FILE/

至此，我们的FreeRTOS基本移植完成，下面是测试的时候了。

修改main.c
~~~~~~~~

我们将原来裸机工程里面main.c的文件内容全部删除，新增如下内容，具体见代码清单11‑4。

代码清单11‑4 main.c文件内容

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief FreeRTOS 3.0 + STM32 工程模版

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

23 #include" FreeRTOS.h"

24 #include" task.h"

25

26

27 /\*

28 \\*

29 \* 变量

30 \\*

31 \*/

32

33

34 /\*

35 \\*

36 \* 函数声明

37 \\*

38 \*/

39

40

41

42 /\*

43 \\*

44 \* main 函数

45 \\*

46 \*/

47 /*\*

48 \* @brief 主函数

49 \* @param 无

50 \* @retval 无

51 \*/

52 int main(void)

53 {

54 /*暂时没有在main任务里面创建任务应用任务 \*/

55 }

56

57

58 /END OF FILE/

下载验证
~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），一看，啥现象都没有，一脸懵逼，我说，你急个肾，目前我们还没有在main任务里面创建应用任务，但是系统是已经跑起来了，只有默认的空闲任务和main任务。要想看现象，得自己在ma
in创建里面应用任务，如果创建任务，请看下一章“创建任务”。

.. |portin002| image:: media\portin002.png
   :width: 4.00649in
   :height: 2.36077in
.. |portin003| image:: media\portin003.png
   :width: 5.76806in
   :height: 4.42982in
.. |portin004| image:: media\portin004.png
   :width: 5.76806in
   :height: 2.56465in
.. |portin005| image:: media\portin005.png
   :width: 2.74395in
   :height: 2.42857in
.. |portin006| image:: media\portin006.png
   :width: 3.76357in
   :height: 2.27273in
.. |portin007| image:: media\portin007.png
   :width: 5.71038in
   :height: 1.9992in
.. |portin008| image:: media\portin008.png
   :width: 5.52597in
   :height: 2.87623in
.. |portin009| image:: media\portin009.png
   :width: 5.76806in
   :height: 2.32085in
.. |portin010| image:: media\portin010.png
   :width: 1.88961in
   :height: 1.5707in
.. |portin011| image:: media\portin011.png
   :width: 5.66234in
   :height: 1.09005in
.. |portin012| image:: media\portin012.png
   :width: 5.76806in
   :height: 2.73178in
.. |portin013| image:: media\portin013.png
   :width: 4.86384in
   :height: 1.98052in
.. |portin014| image:: media\portin014.png
   :width: 4.62338in
   :height: 1.97156in
.. |portin015| image:: media\portin015.png
   :width: 4.14935in
   :height: 1.65749in
.. |portin016| image:: media\portin016.png
   :width: 4.57526in
   :height: 2.27273in
.. |portin018| image:: media\portin017.png
   :width: 2.27806in
   :height: 4.25325in
.. |portin018| image:: media\portin018.png
   :width: 5.76806in
   :height: 4.4492in
.. |portin019| image:: media\portin019.png
   :width: 5.76806in
   :height: 2.56304in
.. |portin020| image:: media\portin020.png
   :width: 5.76806in
   :height: 2.90443in
