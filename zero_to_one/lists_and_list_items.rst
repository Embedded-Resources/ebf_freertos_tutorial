.. vim: syntax=rst

数据结构—列表与列表项讲解
-------------

在FreeRTOS中存在着大量的基础数据结构列表和列表项的操作，要想读懂FreeRTOS的源码或者从0到1开始实现FreeRTOS，就必须弄懂列表和列表项的操作，其实也没那么难。

列表和列表项是直接从FreeRTOS源码的注释中的list 和 list item翻译过来的，其实就是对应我们C语言当中的链表和节点，在后续的讲解中，我们说的链表就是列表，节点就是列表项。

C语言链表简介
~~~~~~~

链表作为C语言中一种基础的数据结构，在平时写程序的时候用的并不多，但在操作系统里面使用的非常多。链表就好比一个圆形的晾衣架，具体见图4‑1，晾衣架上面有很多钩子，钩子首尾相连。链表也是，链表由节点组成，节点与节点之间首尾相连。

晾衣架的钩子本身不能代表很多东西，但是钩子本身却可以挂很多东西。同样，链表也类似，链表的节点本身不能存储太多东西，或者说链表的节点本来就不是用来存储大量数据的，但是节点跟晾衣架的钩子一样，可以挂很多数据。

|listsa002|

图4‑1圆形晾衣架

链表分为单向链表和双向链表，单向链表很少用，使用最多的还是双向链表。

单向链表
^^^^

链表的定义
'''''

单向链表示意图具体见图4‑2。该链表中共有n个节点，前一个节点都有一个箭头指向后一个节点，首尾相连，组成一个圈。

|listsa003|

图4‑2单向链表

节点本身必须包含一个节点指针，用于指向后一个节点，除了这个节点指针是必须有的之外，节点本身还可以携带一些私有信息，怎么携带？

节点都是一个自定义类型的数据结构，在这个数据结构里面可以有单个的数据、数组、指针数据和自定义的结构体数据类型等等信息，具体见代码清单4‑1。

代码清单4‑1节点结构体定义

1 struct node

2 {

3 struct node \*next; /\* 指向链表的下一个节点 \*/

4 char data1; /\* 单个的数据 \*/

5 unsigned char array[]; /\* 数组 \*/

6 unsigned long \*prt /\* 指针数据 \*/

7 struct userstruct data2; /*自定义结构体类型数据 \*/

8 /\* ......
\*/

9 }

在代码清单4‑1除了struct node \*next 这个节点指针之外，剩下的成员都可以理解为节点携带的数据，但是这种方法很少用。通常的做法是节点里面只包含一个用于指向下一个节点的指针。要通过链表存储的数据内嵌一个节点即可，这些要存储的数据通过这个内嵌的节点即可挂接到链表中，就好像晾衣架的钩子一
样，把衣服挂接到晾衣架中，具体的伪代码实现见代码清单4‑2，具体的示意图见图4‑3。

代码清单4‑2节点内嵌在一个数据结构中

1 /\* 节点定义 \*/

2 struct node

3 {

4 struct node \*next; /\* 指向链表的下一个节点 \*/

5 }

6

7 struct userstruct

8 {

9 /\* 在结构体中，内嵌一个节点指针，通过这个节点将数据挂接到链表 \*/

10 struct node \*next;

11 /\* 各种各样......，要存储的数据 \*/

12 }

|listsa004|

图4‑3节点内嵌在一个数据结构中

链表的操作
'''''

链表最大的作用是通过节点把离散的数据链接在一起，组成一个表，这大概就是链表的字面解释了吧。链表常规的操作就是节点的插入和删除，为了顺利的插入，通常一条链表我们会人为地规定一个根节点，这个根节点称为生产者。通常根节点还会有一个节点计数器，用于统计整条链表的节点个数，具体见图4‑4中的root_node
。

|listsa005|

图4‑4带根节点的链表

有关链表节点的删除和操作的代码讲解这里先略过，具体的可参考本章接下来的“FreeRTO中链表的实现”小节，在这个小节里面会有非常详细的讲解，这里我们先建立概念为主。

双向链表
^^^^

双向链表与单向链表的区别就是节点中有两个节点指针，分别指向前后两个节点，其他完全一样。有关双向链表的文字描述参考单向链表小节即可，有关双向链表的示意图具体见图4‑5。

|listsa006|

图4‑5双向链表

链表与数组的对比
^^^^^^^^

在很多公司的嵌入式面试中，通常会问到链表和数组的区别。在C语言中，链表与数组确实很像，两者的示意图具体见图4‑6，这里以双向链表为例。

|listsa007|

图4‑6链表与数组的对比

链表是通过节点把离散的数据链接成一个表，通过对节点的插入和删除操作从而实现对数据的存取。而数组是通过开辟一段连续的内存来存储数据，这是数组和链表最大的区别。数组的每个成员对应链表的节点，成员和节点的数据类型可以是标准的C类型或者是用户自定义的结构体。数组有起始地址和结束地址，而链表是一个圈，没有头和
尾之分，但是为了方便节点的插入和删除操作会人为的规定一个根节点。

FreeRTOS中链表的实现
~~~~~~~~~~~~~~

FreeRTOS中与链表相关的操作均在list.h和list.c这两个文件中实现，list.h第一次使用需要在include文件夹下面新建然后添加到工程freertos/source这个组文件，list.c第一次使用需要在freertos文件夹下面新建然后添加到工程freertos/source这个
组文件。

实现链表节点
^^^^^^

定义链表节点数据结构
''''''''''

链表节点的数据结构在list.h中定义，具体实现见代码清单4‑3，节点示意图具体见图4‑7。

代码清单4‑3链表节点数据结构定义

1 struct xLIST_ITEM

2 {

3 TickType_t xItemValue; /\* 辅助值，用于帮助节点做顺序排列 \*/**(1)**

4 struct xLIST_ITEM \* pxNext; /\* 指向链表下一个节点 \*/**(2)**

5 struct xLIST_ITEM \* pxPrevious; /\* 指向链表前一个节点 \*/**(3)**

6 void \* pvOwner; /\* 指向拥有该节点的内核对象，通常是TCB \*/**(4)**

7 void \* pvContainer; /\* 指向该节点所在的链表 \*/**(5)**

8 };

9 typedefstruct xLIST_ITEM ListItem_t; /\* 节点数据类型重定义 \*/**(6)**

|listsa008|

图4‑7节点示意图

代码清单4‑3\ **(1)**\ ：一个辅助值，用于帮助节点做顺序排列。该辅助值的数据类型为TickType_t，在FreeRTOS中，凡是涉及数据类型的地方，FreeRTOS都会将标准的C数据类型用typedef 重新取一个类型名。这些经过重定义的数据类型放在portmacro.h（portma
cro.h第一次使用需要在include文件夹下面新建然后添加到工程freertos/source这个组文件）这个头文件，具体见代码清单4‑4。代码清单4‑4中除了TickType_t外，其他数据类型重定义是本章后面内容需要使用到，这里统一贴出来，后面将不再赘述。

代码清单4‑4portmacro.h 文件中的数据类型

1 #ifndef PORTMACRO_H

2 #define PORTMACRO_H

3

4 #include"stdint.h"

5 #include"stddef.h"

6

7

8 /\* 数据类型重定义 \*/

9 #define portCHAR char

10 #define portFLOAT float

11 #define portDOUBLE double

12 #define portLONG long

13 #define portSHORT short

14 #define portSTACK_TYPE uint32_t

15 #define portBASE_TYPE long

16

17 typedef portSTACK_TYPE StackType_t;

18 typedeflong BaseType_t;

19 typedefunsigned long UBaseType_t;

20

**21 #if( configUSE_16_BIT_TICKS == 1 )(1)**

22 typedefuint16_t TickType_t;

23 #define portMAX_DELAY ( TickType_t ) 0xffff

24 #else

**25 typedefuint32_t TickType_t;**

26 #define portMAX_DELAY ( TickType_t ) 0xffffffffUL

27 #endif

28

29 #endif/\* PORTMACRO_H \*/

代码清单4‑4\ **(1)**\ ：TickType_t具体表示16位还是32位，由configUSE_16_BIT_TICKS这个宏决定，当该宏定义为1时，TickType_t为16位，否则为32位。该宏在在FreeRTOSConfig.h（FreeRTOSConfig.h第一次使用需要在inc
lude文件夹下面新建然后添加到工程freertos/source这个组文件）中默认定义为0，具体实现见代码清单4‑5，所以TickType_t表示32位。

代码清单4‑5configUSE_16_BIT_TICKS宏定义

1 #ifndef FREERTOS_CONFIG_H

2 #define FREERTOS_CONFIG_H

3

**4 #define configUSE_16_BIT_TICKS 0**

5

6 #endif/\* FREERTOS_CONFIG_H \*/

代码清单4‑3\ **(2)**\ ：用于指向链表下一个节点。

代码清单4‑3\ **(3)**\ ：用于指向链表前一个节点。

代码清单4‑3\ **(4)**\ ：用于指向该节点的拥有者，即该节点内嵌在哪个数据结构中，属于哪个数据结构的一个成员。

代码清单4‑3\ **(5)**\ ：用于指向该节点所在的链表，通常指向链表的根节点。

代码清单4‑3\ **(6)**\ ：节点数据类型重定义。

链表节点初始化
'''''''

链表节点初始化函数在list.c中实现，具体实现见代码清单4‑6。

代码清单4‑6链表节点初始化

1 void vListInitialiseItem( ListItem_t \* const pxItem )

2 {

3 /\* 初始化该节点所在的链表为空，表示节点还没有插入任何链表 \*/

4 pxItem->pvContainer = NULL;\ **(1)**

5 }

代码清单4‑6\ **(1)**\ ：链表节点ListItem_t总共有5个成员，但是初始化的时候只需将pvContainer初始化为空即可，表示该节点还没有插入到任何链表。一个初始化好的节点示意图具体见图4‑8。

|listsa009|

图4‑8节点初始化

实现链表根节点
^^^^^^^

定义链表根节点数据结构
'''''''''''

链表根节点的数据结构在list.h中定义，具体实现见代码清单4‑7，根节点示意图具体见图4‑7。

代码清单4‑7链表根节点数据结构定义

1 typedefstruct xLIST

2 {

3 UBaseType_t uxNumberOfItems; /\* 链表节点计数器 \*/**(1)**

4 ListItem_t \* pxIndex; /\* 链表节点索引指针 \*/**(2)**

5 MiniListItem_t xListEnd; /\* 链表最后一个节点 \*/**(3)**

6 } List_t;

|listsa010|

代码清单4‑8根节点示意图

代码清单4‑7\ **(1)**\ ：链表节点计数器，用于表示该链表下有多少个节点，根节点除外。

代码清单4‑7\ **(2)**\ ：链表节点索引指针，用于遍历节点。

代码清单4‑7\ **(3)**\ ：链表最后一个节点。我们知道，链表是首尾相连的，是一个圈，首就是尾，尾就是首，这里从字面上理解就是链表的最后一个节点，实际也就是链表的第一个节点，我们称之为生产者。该生产者的数据类型是一个精简的节点，也在list.h中定义，具体实现见。

代码清单4‑9链表精简节点结构体定义

1 struct xMINI_LIST_ITEM

2 {

3 TickType_t xItemValue; /\* 辅助值，用于帮助节点做升序排列 \*/

4 struct xLIST_ITEM \* pxNext; /\* 指向链表下一个节点 \*/

5 struct xLIST_ITEM \* pxPrevious; /\* 指向链表前一个节点 \*/

6 };

7 typedefstruct xMINI_LIST_ITEM MiniListItem_t; /\* 精简节点数据类型重定义 \*/

链表根节点初始化
''''''''

链表节点初始化函数在list.c中实现，具体实现见代码清单4‑10，初始化好的根节点示意图具体见。

代码清单4‑10链表根节点初始化

1 void vListInitialise( List_t \* const pxList )

2 {

3 /\* 将链表索引指针指向最后一个节点 \*/**(1)**

4 pxList->pxIndex = ( ListItem_t \* ) &( pxList->xListEnd );

5

6 /\* 将链表最后一个节点的辅助排序的值设置为最大，确保该节点就是链表的最后节点 \*/**(2)**

7 pxList->xListEnd.xItemValue = portMAX_DELAY;

8

9 /\* 将最后一个节点的pxNext和pxPrevious指针均指向节点自身，表示链表为空 \*/**(3)**

10 pxList->xListEnd.pxNext = ( ListItem_t \* ) &( pxList->xListEnd );

11 pxList->xListEnd.pxPrevious = ( ListItem_t \* ) &( pxList->xListEnd );

12

13 /\* 初始化链表节点计数器的值为0，表示链表为空 \*/**(4)**

14 pxList->uxNumberOfItems = ( UBaseType_t ) 0U;

15 }

|listsa011|

图4‑9根节点初始化

代码清单4‑10 **(1)**\ ：将链表索引指针指向最后一个节点，即第一个节点，或者第零个节点更准确，因为这个节点不会算入节点计数器的值。

代码清单4‑10 **(2)**\ ：将链表最后（也可以理解为第一）一个节点的辅助排序的值设置为最大，确保该节点就是链表的最后节点（也可以理解为第一）。

代码清单4‑10 **(3)**\ ：将最后一个节点（也可以理解为第一）的pxNext和pxPrevious指针均指向节点自身，表示链表为空。

代码清单4‑10 **(4)**\ ：初始化链表节点计数器的值为0，表示链表为空。

将节点插入到链表的尾部
'''''''''''

将节点插入到链表的尾部（可以理解为头部）就是将一个新的节点插入到一个空的链表，具体代码实现见代码清单4‑11，插入过程的示意图见图4‑10。

代码清单4‑11将节点插入到链表的尾部

1 void vListInsertEnd( List_t \* const pxList, ListItem_t \* const pxNewListItem )

2 {

3 ListItem_t \* const pxIndex = pxList->pxIndex;

4

5 pxNewListItem->pxNext = pxIndex;\ **①**

6 pxNewListItem->pxPrevious = pxIndex->pxPrevious;\ **②**

7 pxIndex->pxPrevious->pxNext = pxNewListItem;\ **③**

8 pxIndex->pxPrevious = pxNewListItem;\ **④**

9

10 /\* 记住该节点所在的链表 \*/

11 pxNewListItem->pvContainer = ( void \* ) pxList; **⑤**

12

13 /\* 链表节点计数器++ \*/

14 ( pxList->uxNumberOfItems )++; **⑥**

15 }

|listsa012|

图4‑10将节点插入到链表的尾部

将节点按照升序排列插入到链表
''''''''''''''

将节点按照升序排列插入到链表，如果有两个节点的值相同，则新节点在旧节点的后面插入，具体实现见代码清单4‑12。

代码清单4‑12将节点按照升序排列插入到链表

1 void vListInsert( List_t \* const pxList, ListItem_t \* const pxNewListItem )

2 {

3 ListItem_t \*pxIterator;

4

5 /\* 获取节点的排序辅助值 \*/

6 const TickType_t xValueOfInsertion = pxNewListItem->xItemValue;\ **(1)**

7

8 /\* 寻找节点要插入的位置 \*/**(2)**

9 if ( xValueOfInsertion == portMAX_DELAY )

10 {

11 pxIterator = pxList->xListEnd.pxPrevious;

12 }

13 else

14 {

15 for ( pxIterator = ( ListItem_t \* ) &( pxList->xListEnd );

16 pxIterator->pxNext->xItemValue <= xValueOfInsertion;

17 pxIterator = pxIterator->pxNext )

18 {

19 /\* 没有事情可做，不断迭代只为了找到节点要插入的位置 \*/

20 }

21 }

22 /\* 根据升序排列，将节点插入 \*/**(3)**

23 pxNewListItem->pxNext = pxIterator->pxNext; **①**

24 pxNewListItem->pxNext->pxPrevious = pxNewListItem; **②**

25 pxNewListItem->pxPrevious = pxIterator; **③**

26 pxIterator->pxNext = pxNewListItem; **④**

27

28 /\* 记住该节点所在的链表 \*/

29 pxNewListItem->pvContainer = ( void \* ) pxList; **⑤**

30

31 /\* 链表节点计数器++ \*/

32 ( pxList->uxNumberOfItems )++; **⑥**

33 }

|listsa013|

图4‑11将节点按照升序排列插入到链表

代码清单4‑12\ **(1)**\ ：获取节点的排序辅助值。

代码清单4‑12\ **(2)**\ ：根据节点的排序辅助值，找到节点要插入的位置，按照升序排列。

代码清单4‑12\ **(3)**\ ：按照升序排列，将节点插入到链表。假设将一个节点排序辅助值是2的节点插入到有两个节点的链表中，这两个现有的节点的排序辅助值分别是1和3，那么插入过程的示意图具体见图4‑11。

将节点从链表删除
''''''''

将节点从链表删除具体实现见代码清单4‑13。假设将一个有三个节点的链表中的中间节点节点删除，删除操作的过程示意图具体可见图4‑12。

代码清单4‑13将节点从链表删除

1 UBaseType_t uxListRemove( ListItem_t \* const pxItemToRemove )

2 {

3 /\* 获取节点所在的链表 \*/

4 List_t \* const pxList = ( List_t \* ) pxItemToRemove->pvContainer;

5 /\* 将指定的节点从链表删除*/

6 pxItemToRemove->pxNext->pxPrevious = pxItemToRemove->pxPrevious;\ **①**

7 pxItemToRemove->pxPrevious->pxNext = pxItemToRemove->pxNext;\ **②**

8

9 /*调整链表的节点索引指针 \*/

10 if ( pxList->pxIndex == pxItemToRemove )

11 {

12 pxList->pxIndex = pxItemToRemove->pxPrevious;

13 }

14

15 /\* 初始化该节点所在的链表为空，表示节点还没有插入任何链表 \*/

16 pxItemToRemove->pvContainer = NULL; **③**

17

18 /\* 链表节点计数器-- \*/

19 ( pxList->uxNumberOfItems )--; **④**

20

21 /\* 返回链表中剩余节点的个数 \*/

22 return pxList->uxNumberOfItems;

23 }

|listsa014|

图4‑12将节点从链表删除

节点带参宏小函数
''''''''

在list.h中，还定义了各种各样的带参宏，方便对节点做一些简单的操作，具体实现见代码清单4‑14节点带参宏小函数。

代码清单4‑14节点带参宏小函数

1 /\* 初始化节点的拥有者 \*/

2 #define listSET_LIST_ITEM_OWNER( pxListItem, pxOwner )\\

3 ( ( pxListItem )->pvOwner = ( void \* ) ( pxOwner ) )

4

5 /\* 获取节点拥有者 \*/

6 #define listGET_LIST_ITEM_OWNER( pxListItem )\\

7 ( ( pxListItem )->pvOwner )

8

9 /\* 初始化节点排序辅助值 \*/

10 #define listSET_LIST_ITEM_VALUE( pxListItem, xValue )\\

11 ( ( pxListItem )->xItemValue = ( xValue ) )

12

13 /\* 获取节点排序辅助值 \*/

14 #define listGET_LIST_ITEM_VALUE( pxListItem )\\

15 ( ( pxListItem )->xItemValue )

16

17 /\* 获取链表根节点的节点计数器的值 \*/

18 #define listGET_ITEM_VALUE_OF_HEAD_ENTRY( pxList )\\

19 ( ( ( pxList )->xListEnd ).pxNext->xItemValue )

20

21 /\* 获取链表的入口节点 \*/

22 #define listGET_HEAD_ENTRY( pxList )\\

23 ( ( ( pxList )->xListEnd ).pxNext )

24

25 /\* 获取节点的下一个节点 \*/

26 #define listGET_NEXT( pxListItem )\\

27 ( ( pxListItem )->pxNext )

28

29 /\* 获取链表的最后一个节点 \*/

30 #define listGET_END_MARKER( pxList )\\

31 ( ( ListItem_t const \* ) ( &( ( pxList )->xListEnd ) ) )

32

33 /\* 判断链表是否为空 \*/

34 #define listLIST_IS_EMPTY( pxList )\\

35 ( ( BaseType_t ) ( ( pxList )->uxNumberOfItems == ( UBaseType_t ) 0 ) )

36

37 /\* 获取链表的节点数 \*/

38 #define listCURRENT_LIST_LENGTH( pxList )\\

39 ( ( pxList )->uxNumberOfItems )

40

41 /\* 获取链表第一个节点的OWNER，即TCB \*/

42 #define listGET_OWNER_OF_NEXT_ENTRY( pxTCB, pxList ) \\

43 { \\

44 List_t \* const pxConstList = ( pxList ); \\

45 /\* 节点索引指向链表第一个节点 \*/ \\

46 ( pxConstList )->pxIndex = ( pxConstList )->pxIndex->pxNext; \\

47 /\* 这个操作有啥用？ \*/\\

48 if( ( void \* ) ( pxConstList )->pxIndex == ( void \* ) &( ( pxConstList )->xListEnd ) ) \\

49 { \\

50 ( pxConstList )->pxIndex = ( pxConstList )->pxIndex->pxNext; \\

51 } \\

52 /\* 获取节点的OWNER，即TCB \*/\\

53 ( pxTCB ) = ( pxConstList )->pxIndex->pvOwner; \\

54 }

链表节点插入实验实验
~~~~~~~~~~

我们新建一个根节点（也可以理解为链表）和三个普通节点，然后将这三个普通节点按照节点的排序辅助值做升序排列插入到链表中，具体代码见代码清单4‑15。

代码清单4‑15链表节点插入实验

1 /\*

2 \\*

3 \* 包含的头文件

4 \\*

5 \*/

6 #include"list.h"

7

8 /\*

9 \\*

10 \* 全局变量

11 \\*

12 \*/

13

14 /\* 定义链表根节点 \*/

15 struct xLIST List_Test;\ **(1)**

16

17 /\* 定义节点 \*/

18 struct xLIST_ITEM List_Item1;\ **(2)**

19 struct xLIST_ITEM List_Item2;

20 struct xLIST_ITEM List_Item3;

21

22

23

24 /\*

25 \\*

26 \* main函数

27 \\*

28 \*/

29 /\*

30int main(void)

31{

32

33/\* 链表根节点初始化 \*/

34 vListInitialise( &List_Test );\ **(3)**

35

36/\* 节点1初始化 \*/

37 vListInitialiseItem( &List_Item1 );\ **(4)**

38 List_Item1.xItemValue = 1;

39

40/\* 节点2初始化 \*/

41 vListInitialiseItem( &List_Item2 );

42 List_Item2.xItemValue = 2;

43

44/\* 节点3初始化 \*/

45 vListInitialiseItem( &List_Item3 );

46 List_Item3.xItemValue = 3;

47

48/\* 将节点插入链表，按照升序排列 \*/**(5)**

49 vListInsert( &List_Test, &List_Item2 );

50 vListInsert( &List_Test, &List_Item1 );

51 vListInsert( &List_Test, &List_Item3 );

52

53for (;;)

54 {

55/\* 啥事不干 \*/

56 }

57}

代码清单4‑15\ **(1)**\ ：定义链表根节点，有根了，节点才能在此基础上生长。

代码清单4‑15\ **(2)**\ ：定义3个普通节点。

代码清单4‑15\ **(3)**\ ：链表根节点初始化，初始化完毕之后，根节点示意图见图4‑13。

|listsa015|

图4‑13链表根节点初始化

代码清单4‑15\ **(4)**\ ：节点初始化，初始化完毕之后节点示意图见图4‑14，其中xItemValue等于你的初始化值。

|listsa016|

图4‑14链表节点初始化

代码清单4‑15\ **(5)**\ ：将节点按照他们的排序辅助值做升序排列插入到链表，插入完成后链表的示意图见图4‑15。

|listsa017|

图4‑15节点按照排序辅助值做升序排列插入到链表

实验现象
^^^^

实验现象如图4‑15所示，但这好像是我得出的结论，是否有准确的数据支撑？有的，我们可以通过软件仿真来证实。

将程序编译好之后，点击调试按钮，然后全速运行，再然后把List_Test、List_Item1、List_Item2和List_Item3这四个全局变量添加到观察窗口，然后查看这几个数据结构中pxNext和pxPrevious的值即可证实图图4‑15是正确的，具体的仿真数据见图4‑16。

|listsa018|

图4‑16节点按照排序辅助值做升序排列插入到链表软件仿真数据

.. |listsa002| image:: media\listsa002.jpeg
   :width: 1.9422in
   :height: 1.9422in
.. |listsa003| image:: media\listsa003.png
   :width: 4.18497in
   :height: 0.67473in
.. |listsa004| image:: media\listsa004.png
   :width: 5.76806in
   :height: 1.30449in
.. |listsa005| image:: media\listsa005.png
   :width: 5.76806in
   :height: 0.96802in
.. |listsa006| image:: media\listsa006.png
   :width: 5.76806in
   :height: 1.06482in
.. |listsa007| image:: media\listsa007.png
   :width: 5.76806in
   :height: 1.44802in
.. |listsa008| image:: media\listsa008.png
   :width: 2.20969in
   :height: 2.42406in
.. |listsa009| image:: media\listsa009.png
   :width: 3.09091in
   :height: 2.15648in
.. |listsa010| image:: media\listsa010.png
   :width: 3.41558in
   :height: 2.59331in
.. |listsa011| image:: media\listsa011.png
   :width: 3.07792in
   :height: 2.15775in
.. |listsa012| image:: media\listsa012.png
   :width: 4.16883in
   :height: 2.55437in
.. |listsa013| image:: media\listsa013.png
   :width: 5.76806in
   :height: 2.08157in
.. |listsa014| image:: media\listsa014.png
   :width: 5.76806in
   :height: 2.10828in
.. |listsa015| image:: media\listsa015.png
   :width: 2.46104in
   :height: 1.8133in
.. |listsa016| image:: media\listsa016.png
   :width: 2.61688in
   :height: 1.6195in
.. |listsa017| image:: media\listsa017.png
   :width: 5.76806in
   :height: 2.02082in
.. |listsa018| image:: media\listsa018.png
   :width: 4.64286in
   :height: 3.28117in
