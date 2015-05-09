---
layout: post
title: UE4中的行为树
---

## 1. 什么是行为树
行为树是通过层次节点控制AI实体的流程决策，叶子节点是实际执行的命令。通常用于替代拥有超多状态的有限状态机(FSM)。

## 2. UE4中的行为树
UE4的行为树是基于事件驱动的，避免每一帧都做很多无用的判断。通过装饰节点来实现条件判断。

### 2.1 根(Root)

顾名思义，是行为树的执行起始点，每个行为树有一个。无法绑定Decorator或者Service。

### 2.2 组合(Composite)

用于定义一个分支的根以及分支的基础规则.  
Composite又分为三个子类:

* 选择器(Selector)   
从左到右执行它的孩子节点，如果某个孩子成功，那么停止执行。如果选择器的孩子成功，那么选择器成功；如果选择器的孩子都失败，那么选择器失败。

属性  | 描述
------------- | -------------
Node Name  | 在行为树图形中节点显示的名字	

* 顺序(Sequence)  
从左到右执行它的孩子节点，如果某个孩子失败了，那么停止执行。如果选择器的汉子成功，那么选择器成功；如果选择器的孩子一个失败，那么选择器失败。

属性  | 描述
------------- | -------------
Node Name  | 在行为树图形中节点显示的名字	

* 简单并行(Simple Parallel)   
允许一个单一的主Task节点和整个行为树一起执行。当主Task完成时，通过设置Finish Mode决定节点立刻完成，终止执行第二个行为树，或者延迟直到第二个行为树完成。

属性  | 描述
------------- | -------------
Finish Mode | Immediate 一旦主Task完成，后台树会被终止；Delayed 主Task完成后，后台树会继续执行直到完成。
Node Name  | 在行为树图形中节点显示的名字	

### 2.3 任务(Task)

行为树的叶子节点，做具体事情的节点。  
常用的Task有:

* 制造噪音(Make Noise)  
如果被操作的pawn使用了PawnNoiseEmitter组件，那么这个任务会使它产生噪音(发出消息），其它使用了PawnSensing组件的pawns会收听到（接收消息）。

属性  | 描述
------------- | -------------
Loudness  | 发出声音的音高
Node Name  | 在行为树图形中节点显示的名字	
	
* 移动到(Move To)  
此节点会使使用Character Movement组件的pawn移动，移动过程会使用Blackboard Key中的NavMesh数据。

属性  | 描述
------------- | -------------
Acceptable Radius |
Filter Class |
Allow Strafe |
Blackboard Key |
Node Name |

* 发出声音(Play Sound)  
播放Sound to Play属性设定的声音

属性  | 描述
------------- | -------------
Sound to Play |
Node Name |

* 执行行为(Run Behavior)  
执行另一个行为树


属性  | 描述
------------- | -------------
Behavior Asset |
Node Name |


* 执行EQS查询(Run EQS Query)   
EQS的全称是Enviroment Query System。


属性  | 描述
------------- | -------------
Query Template |
Query Params |
Blackboard Key|
Node Name |


* 等待(Wait)

在此节点上等待，指导设定的Wait Time到期


属性  | 描述
------------- | -------------
Wait Time |
Node Name |

* 等待Blackboard时间（Wait Blackboard Time)   
跟前一个Wait类似，唯一不同的是等待时间通过Blackboard获取。


属性  | 描述
------------- | -------------
Blackboard Key |
Node Name |


### 2.4 装饰(Decorator)

也称为条件。跟另一个节点绑定决策是否执行树的分支或者单个节点。

* Blackboard   
检查看是否设定了指定的Blackboard Key








属性  | 描述
------------- | -------------
Notify Observer|  <ul><li>On Result Change</li><li>On Value Change</li></ul>
Observer | <ul><li>None</li><li>Self</li><li>Lower Priority</li><li>Both</li></ul>
Key Query | <ul><li>Is Set</li><li>Is Not Set</li></ul>
 Blackboard Key|
 Node Name |

* Compare Blackboard Entries  
该节点会比较两个Blackboard Key的值，依据结果来阻塞或者允许节点的执行

属性  | 描述
------------- | -------------
Operator|  <ul><li>Is Equal To</li><li>Is Not Equal To</li></ul>
Blackboard Key A |
Blackboard Key B|
Observer Aborts | <ul><li>None</li><li>Self</li><li>Lower Priority</li><li>Both</li></ul>
 Node Name |
 
* Composite
组合其它的装饰节点，可以处理AND/OR/NOT的关系
* Cone Check   
The Cone Check decorator takes in 3 vector keys: the first for the location to start the cone, the second to define the direction the cone points, and the third for the location to check if it is inside the cone. You define the angle of the cone by using the Cone Half Angle property.

* Cooldown

该节点会锁住节点或者分支的执行，直到冷却时间达到。

* Does Path Exist

检查两个指定的向量之间是否有路径：Blackboard Key A和Blackboard Key B

* Force Success  

改变节点的结果为成功

* Keep in Cone
* Loop

循环节点或者分支指定次数，或者永久

* Reached Move Goal

节点检查pawn(使用了Character Movement组件)的路径是否已经到达目标。

* Time Limit
该节点给节点或者分支设置超时时间，每次节点获得焦点定时器会被重置。


### 2.5 服务(Service)

跟Composite绑定，只要分支被执行，它就会以预定义的频率被执行。被常用于检测或者更新黑板（Blackboard)。





 

## UE4中代码分析（未完成）


## 参考
1. [Behavior Trees Nodes Reference](https://docs.unrealengine.com/latest/INT/Engine/AI/BehaviorTrees/NodeReference/index.html)
2. [How Unreal Engine 4 Behavior Trees Differ](https://docs.unrealengine.com/latest/INT/Engine/AI/BehaviorTrees/HowUE4BehaviorTreesDiffer/index.html)
3. [Behavior trees for AI: How they work](http://www.gamasutra.com/blogs/ChrisSimpson/20140717/221339/Behavior_trees_for_AI_How_they_work.php)