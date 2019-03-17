---
title: "IBM实习总结"
date: 2013-04-01
type: "posts"
draft: false
---

今天上午和新来的实习生交接了自己的工作，中午约同事们一起吃了午饭，之后很快办完了离职手续，自己为期6个多月的IBM实习也画上了句号，不管是不是完美，对于我自己来说，大学四年的第一份实习无疑对我价值颇高。这篇文章我主要说说我在IBM实习的经历以及感受。

### 实习时间
---------------
我出去实习的时间比较晚，根据学院规定暑假就可以开实习，时间不能少于6个月，所以最好应该在4，5月份找实习，而我在4，5月份却忙于备考GRE和Toefl，自然也错过了找实习的黄金时间。暑假结束后，我才开始计划找实习。开始一心想着去互联网公司，也正因为一直在等待这样的机会而浪费了不少时间。直达9月份，刷[小百合](http://bbs.nju.edu.cn/)看到一个不错的实习机会，也就是接下来6个多月我所在的IBM JTC部门。当时心动的主要原因是这个实习职位所在的Team是做JVM的，也是自己的兴趣方向所在，所以果断投了简历。

### 面试
---------------
IBM的面试分为技术面和英文面，可能当时急缺实习生，所以面试安排得很紧凑。自己也比较幸运，因为Team里有已经毕业的学长Ray，多少会有加分：）。因为是JVM小组，所以技术面都是关于Java的，比如Java多线程，IO，容器类以及反射机制等，没有问算法题。现在回想起来，这些知识确实在每天的实习工作中都会有所接触。接下来是英文面，主要面试口语。因为是招聘实习生，所以也没有太高的要求，基本的听说读写熟练就没有问题。最后面试官问我有没有问题想问他的，我进一步问了关于实习职位的工作内容，当时的面试官，也就是后来我的mentor Sanhong，人非常nice，blabla...讲了一大堆，虽然当时也不是很懂，但真心觉得很NB。过了一个周左右，收到面试通过的邮件，几天后搭乘了去上海的动车，开始了我的IBM实习。

### 工作环境
---------------
IBM的工作环境很不错，整个办公室是个很大的开放式的环境，整个部门，从部门老大，到Manager，到实习生都在这里工作。我去的时候领到了一台旧电脑，是的，实习生用的都是旧电脑，不多这也完全不影响开发，因为基本的开发测试都是在云端，通过SecureCRT SSH登录到云端Linux。也正是从这开始彻底喜欢上了Unix/Linux哲学：
> simple and beautiful                        --[Wikipedia](http://en.wikipedia.org/wiki/Unix_philosophy)

对于习惯了IDE的我一开始会有些不适应，不过后来发现在Terminal下工作效率丝毫不逊于IDE。

### 工作内容
---------------
IBM比较注重基础性软件研发，特别在中国成立CDL(China Development Lab)，我所在的部门JTC(Java Technology Center)正式属于CDL，而我所在的小组从事的是JVM的开发。IBM的J9 JVM与Oracle 的Hotspot VM齐名，是两大主流的JVM之一，为IBM许多Java产品提供支持，比如[WebSphere](http://en.wikipedia.org/wiki/IBM_WebSphere)，以及一些开源的产品如[Apache Harmony](http://en.wikipedia.org/wiki/Apache_Harmony)。现在我们team的工作是与加拿大以及印度的同事合作，基于J9VM开发[Multitennancy JVM](http://www.ibm.com/developerworks/library/j-multitenant-java/)，通过在单一的多租户 JVM 中运行多个应用程序，云系统可以加快应用程序的启动时间，并减少其内存占用。这将作为IBM Java8的一个新特性。因为是实习生，所以我的工作大多是于解决Bug，性能调优以及测试相关。我的mentor Sanhong是个技术大牛，人也非常nice，我很庆幸能遇到这样的导师。mentor对我的帮助不仅是技术上的提高，更多的是工作方式的改进，这些东西在学校的绝对学习不到的。

### IBM软件过程管理
--------------------
特别要提到的是IBM的软件过程管理方式，IBM使用敏捷软件开发方式，更具体点儿是Scrum，每两周一次Sprint迭代，每天都会下午选个时间Daily Scrum Meeting，控制在15分钟左右，每个人都必须发言，也包括实习生，向所有成员当面汇报你昨天完成了什么，并且向所有成员承诺你今天要完成什么，同时遇到不能解决的问题也可以提出，每个人回答完成后，要走到黑板前更新自己的 Sprint burn down（Sprint燃尽图）。同时，Team会做到每天至少一个Build，即一个成功编译，可以运行的版本。虽然这些东西在学校也学过，也有实践课程可以体验，但是感觉多少还是是纸上谈兵。如今在算是真正有机会在工作机会中体会Scrum。和其他外企一样，IBM工作语言是英语，虽然平时和同事交流可以用中文，但是邮件以及Message全部都是英文，而且每周一次国际会议也是用英文交流。

### 总结
--------------------
在IBM工作，最重要的是团队合作，虽然平时工作的压力不大，实习生见见“世面”是可以的，但如果真要在技术上有所提升，建议IBM和其他类似的外企可以不用去了，可以去百度，阿里这样的互联网公司，对于锻炼自己的技术应该帮助更大。当然，也可以选择一些创业公司，现在正值互联网蓬勃发展的时候，去小型创业公司，自己可以独当一面，项目经验提升也是必然的。当然小型公司也有自己的缺陷，缺少自己的平台，过多利用现有的技术做产品，对于想从事底层操作系统的基础架构的同学就要重新考虑了。
不过总体来说，通过这次的实习经历，我学习到了不少的东西，不只是技术上，更多是关于工作方式以及团队意识。当然，也第一次去魔都体验了码农的生活，将自己在学校学习的知识利用到了实践当中，赚了自己的第一笔钱。