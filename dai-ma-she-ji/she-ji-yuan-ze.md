# 设计原则

在开始讲设计原则之前，先说说为什么要提这些苦涩无味的知识，相信看这篇文章的人，都是想提升自己，写出更好的的代码。那么什么样的代码是优秀的代码？

最普遍的答案是：**可读性强、可维护性、可扩展性高**

为了实现以上的特点，我们需要借助**面向对象设计**，遵循知名**设计原则**，利用**设计模式**等多种方式来实现。因此本章会介绍一些业界知名的设计原则，希望能给你带来一点思考。

### SOLID原则

#### SRP单⼀职责原则

⼀个类只负责完成⼀个职责或者功能。

#### OCP开闭原则

开闭原则简言而知：对扩展开放、修改关闭。添加⼀个新的功能，应该是通过在已有代码基础上扩展代码（新增模块、类、⽅法、属性等），⽽⾮修改已有代码（修改模 块、类、⽅法、属性等）的⽅式来完成。

#### LSP⾥式替换原则

⼦类对象（object of subtype/derived class）能够替换程序（program）中⽗类对象（object of base/parent class）出现的任 何地⽅，并且保证原来程序的逻辑⾏为（behavior）不变及正确性不被破坏。

#### ISP接⼝隔离原则

客户端不应该强迫依赖它不需要的接⼝

#### DIP依赖 反转原则

### KISS、YAGNI原则

KISS原则：尽量保持简单

YAGNI原则：You Ain’t Gonna Need It 当前不需要的就不要做，不要过度设计

#### DRY原则

不要写重复的代码

