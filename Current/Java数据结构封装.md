> 看了不到三分之一的《算法》之后，学了姿势却无处施展的我，觉得是时候好好深入了解一下Java的集合类实现了，顺便还能复习一下Java，于是就有了这一篇。

### 概述

千言万语，不如一张UML类图:

![Collection类图一](F:\Android\Note\Current\img\Collection类图.png)

一张大概不够，再来一张：

![Collection类图二](F:\Android\Note\Current\img\(补)Map类图.png)

Collection的大致结构已经在图里画得很清楚了，虚线表示实现接口，实线表示继承父类（盗图绝对只是因为我懒），接下来就大致按照_List->Map->Set->Queue_的顺序记一下它们各自的实现。

### 一、List

​	List大概是平时开发中最常见的一种集合类了，它是Collection的直接子类，支持**随机读取**和顺序迭代。下面先从其最常用的子类ArrayList看起。

​	**ArrayList**类如其名，基础数据结构是数组，可以**自动扩充长度**，按照数组角标进行迭代，**线程不安全**，并且支持**快速失败机制**。



> 快速失败机制，则当List的迭代器创建后，以除了Iterator的remove和add方法之外的任何方式修改List中的值，都会使迭代器抛出ConcurrentModificationException，避免因为集合的修改造成之后的不确定错误。

