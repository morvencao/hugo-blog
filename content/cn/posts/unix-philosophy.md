---
title: "Unix哲学"
date: 2013-10-06
type: "post"
draft: false
---


### 两个很老的小故事。

##### 第一个故事。

有一家日本最大的化妆品公司，收到了用户的投诉。用户抱怨买来的肥皂盒是空的。这家公司为了防止再发生这样的事故，很辛苦地发明了一台X光检查器，能够透视每一个出货的肥皂盒。
同样的事故，发生在一家小公司。他们的解决方法是买一台强力的工业电扇，对着肥皂盒猛吹，被吹走的就是空肥皂盒。

##### 第二个故事。

美国太空总署（NASA）发现在太空失重状态下，航天员无法用墨水笔写字。于是，他们花了大量经费，研发出了一种可以在失重状态下写字的太空笔。猜猜看，俄国人是怎么解决的？（答案在本文结尾处。）

### The Philosophy of Unix

很多人在谈"Unix哲学"，也就是开发Unix系统的指导思想。那么什么是Unix哲学

> The Unix philosophy emphasizes building simple, short, clear, modular, and extensible code that can be easily maintained and repurposed by developers other than its creators. The Unix philosophy favors composability as opposed to monolithic design.
>

其实还有其他不同的版本，也就是说不同的人有不同的理解。其中发明管道命令的Doug McIlroy总结了三条，而Eric S. Raymond则在The Art of Unix Programming一书中，一口气总结了17条（[英文版](http://www.faqs.org/docs/artu/ch01s06.html)，[中文版](http://book.csdn.net/bookfiles/34/10034992.shtml)）。
但其实，所有人的总结都离不开一个核心：“简单原则” ----尽量用简单的方法解决问题----是“Unix哲学”的根本原则。这也就是著名的KISS（keep it simple, stupid），意思是"保持简单和笨拙"。
如果想最简单地完成一项编程任务，可以从四个方面入手：

1. 清晰原则。
代码要写得尽量清晰，避免晦涩难懂。清晰的代码不容易崩溃，而且容易理解和维护。重视注释。不为了性能的一丁点提升，而大幅增加技术的复杂性，因为复杂的技术会使得日后的阅读和维护更加艰难。

2. 模块原则。
每个程序只做一件事，不要试图在单个程序中完成多个任务。在程序的内部，面向用户的界面（前端）应该与运算机制（后端）分离，因为前端的变化往往快于后端。

3. 组合原则。
不同的程序之间通过接口相连。接口之间用文本格式进行通信，因为文本格式是最容易处理、最通用的格式。这就意味着尽量不要使用二进制数据进行通信，不要把二进制内容作为输出和输入。

4. 优化原则。
在功能实现之前，不要考虑对它优化。最重要的是让一切先能够运行，其次才是效率。"先求运行，再求正确，最后求快。"（Make it run, then make it right, then make it fast.）90%的功能现在能实现，比100%的功能永远实现不了强。先做出原型，然后找出哪些功能不必实现，那些不用写的代码显然无需优化。目前，最强大的优化工具恐怕是Delete键。

答案是，俄国人用铅笔。：）
